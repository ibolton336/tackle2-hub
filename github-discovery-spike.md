# Spike: discovering applications from a GitHub organisation — the hub side

Answers the four questions in [konveyor/tackle2-ui#3704](https://github.com/konveyor/tackle2-ui/issues/3704)
from inside `tackle2-hub`. Not an implementation plan for a feature that has been
agreed; a set of findings with enough evidence behind them to estimate the work
and to correct a few assumptions the issue makes.

Everything below is against `konveyor/tackle2-hub` at `680736a8`
(`:sparkles: Seed maven CA cert for the java extension (#1145)`), plus
`konveyor/tackle2-addon-platform@main` and `konveyor/tackle2-operator@main`,
both read through the GitHub API rather than cloned.

Measurements in §2 come from a throwaway `net/http` client (no SDK) run against
the live GitHub REST and GraphQL APIs with a classic PAT, so the numbers reflect
what the hub would actually pay rather than what an SDK's docs claim.

---

## Summary

- **A `github` platform kind is cheap and lands in four repositories.** The
  discovery machinery really is kind-agnostic, as the issue says. The work is a
  GitHub provider in `tackle2-addon-platform` (~2 weeks); everything else is
  schema YAML and a `KIND_MAP` entry.
- **A synchronous browse endpoint should not call GitHub from the request
  path.** Not because the hub is the wrong process — it is the right one, and
  there is already precedent for a synchronous third-party REST client in it —
  but because GitHub has no server-side name filter on the org-repos endpoint,
  and the only endpoint that does (`/search/repositories`) is capped at 30
  requests/minute for the whole token and 1,000 results total. Both verified
  below. Any typeahead over a 23,000-repo org must read a local copy. Once you
  accept that, the browse endpoint is a paginated query over a hub table, and
  the thing that fills the table is the same addon that does discovery.
- **PAT first; GitHub App later, and for operational reasons, not throughput.**
  A full 23,000-repo enumeration costs ~230 of a PAT's 5,000 hourly requests.
  Rate limit is not the binding constraint, so the App's 2.5× budget buys
  nothing here. The App's real cost is in the clone path, not the discovery path.
- **One kind per provider, not one `git-forge` kind.** The kind string *is* the
  JSON Schema variant, so one kind means one filter schema and one UI component.
  Share code in the addon, not the kind. GHE needs no kind of its own — it is a
  `Platform.URL`.

Four things in the issue need correcting; they are collected in
[§5](#5-where-the-issues-assumptions-need-correcting). The most consequential is
that **`Platform` has no `coordinates` field** — so "organisation in
`coordinates`" has nowhere to go without a migration.

---

## 1. What a `github` platform kind costs

### 1.1 How `cloudfoundry` is wired, end to end

Five pieces across four repositories.

**The platform record.** `model.Platform`
([`internal/migration/v24/model/platform.go:15`](internal/migration/v24/model/platform.go)):

```go
type Platform struct {
	Model
	Name         string
	Kind         string
	URL          string
	IdentityID   *uint
	Identity     *Identity
	Applications []Application `gorm:"constraint:OnDelete:SET NULL"`
	Tasks        []Task        `gorm:"constraint:OnDelete:CASCADE"`
}
```

`Kind` is a plain string column — no enum, no check constraint, and
`PlatformHandler.Create` ([`internal/api/platform.go:89`](internal/api/platform.go))
does not validate it. A new kind needs no hub code and no migration. The REST
shape is `api.Platform` ([`shared/api/platform.go:4`](shared/api/platform.go)),
converted by `resource.Platform.With`/`.Model`
([`internal/api/resource/platform.go`](internal/api/resource/platform.go)).

**The task.** `application-import` is a `Task` CR
([`internal/k8s/seed/resources/task.yaml:16`](internal/k8s/seed/resources/task.yaml))
carrying `data.action: import`, routed to the `platform` addon by the regex
`^(application-import|application-manifest|asset-generation)$`
([`internal/k8s/seed/resources/addon.yaml:80`](internal/k8s/seed/resources/addon.yaml),
image `quay.io/konveyor/tackle2-addon-platform`).

**The addon.** `konveyor/tackle2-addon-platform`. `cmd/main.go` unpacks the task
secret into `Data{Action, Filter api.Map, …}`; `cmd/action.go` dispatches on
`Action` and selects a provider by kind:

```go
// cmd/action.go
func (r *BaseAction) selectProvider(kind string) (p Provider, err error) {
	switch strings.ToLower(kind) {
	case "cloudfoundry":
		p = &cloudfoundry.Provider{URL: r.platform.URL}
	default:
		err = errors.New("platform kind not supported")
		...
	}
	if r.platform.Identity == nil || r.platform.Identity.ID == 0 {
		return
	}
	id, err := addon.Identity.Decrypted().Get(r.platform.Identity.ID)
	...
	p.Use(id)
}
```

The `Provider` interface is four methods:

```go
type Provider interface {
	Use(identity *api.Identity)
	Fetch(application *api.Application) (m *api.Manifest, err error)
	Find(filter api.Map) (found []api.Application, err error)
	Tag() string
}
```

`cmd/cloudfoundry/provider.go` implements it over
`github.com/konveyor/asset-generation/pkg/providers/discoverers/cloud_foundry`.
`Find` deserialises the task's `Filter` map into a typed `Filter`, lists apps,
matches client-side (including glob matching on names), looks up the coordinates
schema by `addon.Schema.Find("platform", "cloudfoundry", "coordinates")`, and
returns `api.Application` values carrying **only `Name` and `Coordinates`** —
never `Repository`.

`cmd/import.go` then tags each result, sets `app.Platform`, and calls
`addon.Application.Create(&app)` one application at a time, followed by a
per-application manifest `Fetch`.

**The schemas.** Served by `SchemaHandler.Find`
([`internal/api/schema.go:111`](internal/api/schema.go)) on route
`SchemaFindRoute = "/schema" + "/jsd/:domain/:variant/:subject"`
([`shared/api/pkg.go:327`](shared/api/pkg.go)) — so the UI's
`/schema/jsd/platform/{kind}/filter` is exactly `domain=platform`,
`variant={kind}`, `subject=filter`. `jsd.Manager`
([`internal/jsd/manager.go`](internal/jsd/manager.go)) backs it by listing
`tackle.konveyor.io/v1alpha1 Schema` CRs from the hub's namespace and keying
them `domain.variant.subject` (`Manager.Key`, line 104). **The kind is the
schema variant.** That single fact drives the answer to question 4.

The CRs themselves live in **two** places:

| Where | File | Used by |
| --- | --- | --- |
| Operator | `roles/tackle/templates/customresource-schema.yml.j2` | real clusters |
| Hub | [`internal/k8s/seed/resources/jsd.yaml`](internal/k8s/seed/resources/jsd.yaml) | the disconnected simulator only ([`internal/k8s/simulator/client.go:37`](internal/k8s/simulator/client.go)) |

They are hand-duplicated and currently identical. A new kind must be added to
both or dev-mode and production diverge.

**The UI.** A `KIND_MAP` entry in `client/src/app/hooks/usePlatformKindList.ts`
(today a single `cloudfoundry` entry) and a filter-input component. The wizard
renders the filter schema generically.

### 1.2 What a GitHub equivalent needs

| # | Piece | Repository | File |
| --- | --- | --- | --- |
| 1 | `github-filter` + `github-coordinates` Schema CRs | tackle2-operator | `roles/tackle/templates/customresource-schema.yml.j2` |
| 2 | The same two CRs for the simulator | tackle2-hub | `internal/k8s/seed/resources/jsd.yaml` |
| 3 | `github.Provider` implementing `Provider` | tackle2-addon-platform | new `cmd/github/provider.go` |
| 4 | `case "github":` in `selectProvider` | tackle2-addon-platform | `cmd/action.go` |
| 5 | Make `Import` tolerate a provider with no manifest | tackle2-addon-platform | `cmd/import.go` |
| 6 | `KIND_MAP` entry + `filter-input-github.tsx` + i18n | tackle2-ui | `client/src/app/hooks/`, `client/src/app/components/` |

**Nothing is needed in the hub itself** beyond (2), and (2) only affects
disconnected mode. That part of the issue's claim holds up well.

Proposed filter schema (`domain: platform, variant: github, subject: filter`),
mirroring the Cloud Foundry one's style:

```yaml
organizations:   # array, minItems 1 — GitHub org or user logins
visibility:      # enum: all | public | private  (default all)
topics:          # array of strings, AND-ed
names:           # array, each may be a glob — matched client-side, as CF does
includeArchived: # boolean, default false
includeForks:    # boolean, default false
```

`organizations` is the only required field. `visibility` maps to the `type`
query parameter on `GET /orgs/{org}/repos`; the rest is client-side filtering,
which is what the Cloud Foundry provider already does (`Filter.Match`).

Coordinates (`subject: coordinates`) should be `{owner, repo}`. Strictly this is
redundant with `repository.url`, but it keeps `Fetch`/re-sync addressable
without re-parsing URLs and lets the existing filter package query by owner.

Mapping a repo to an application — the part that is genuinely new, because the
Cloud Foundry provider never populates `Repository`:

```go
r := api.Application{Name: repo.FullName}
r.Repository = &api.Repository{
	Kind:   "git",
	URL:    repo.CloneURL,
	Branch: repo.DefaultBranch,
}
r.Coordinates = &api.Document{Schema: schema.Name, Content: api.Map{
	"owner": owner, "repo": repo.Name,
}}
```

`default_branch` and `clone_url` both come back in the list payload, so no
second request per repo. Verified — sample from the live API:

```
sample: name=konveyor/mig-demo-apps default_branch="master" clone_url=https://github.com/konveyor/mig-demo-apps.git private=false
```

Note `master` there: this is exactly the field people get wrong by hand, which
is the strongest argument in the issue.

### 1.3 Three scale problems the import path has at 23,000 applications

These are not blockers, but they are not free either, and the issue does not
mention them.

**a. Application names are globally unique.**
`Application.Name` is `gorm:"index;unique;not null"`
([`internal/migration/v24/model/application.go:15`](internal/migration/v24/model/application.go)).
Repository short names collide across organisations and with applications that
already exist. Use `owner/repo` as the name. Even then, `cmd/import.go` logs a
warning and continues on every create failure — 23,000 repos with a handful of
collisions produces a task report that nobody reads. Collisions should be
counted and summarised, not logged per application.

**b. There is no batch create for applications.**
`BatchHandler.AddRoutes` ([`internal/api/batch.go:19`](internal/api/batch.go))
registers `/batch/tickets` and `/batch/tags` and nothing else. 23,000
applications means 23,000 sequential `POST /applications`, each its own
transaction with a unique-index check. Worth measuring before assuming it needs
fixing — but `/batch/applications` is a ten-line addition following the existing
pattern if it does.

**c. `Import.Run` calls `Fetch` for every created application.**
GitHub has no manifest to fetch, so a GitHub provider's `Fetch` must either
return a recognised "not supported" sentinel or `Import` must skip it by kind.
Left as-is, every one of the 23,000 applications also produces a warning.

---

## 2. Is a synchronous browse endpoint viable?

### 2.1 Measurements

Throwaway `net/http` client, classic PAT, from a laptop on a domestic
connection. Latencies are therefore pessimistic for a cluster, but the request
*counts* and the API's own limits are exact.

**Enumerating an organisation** (`GET /orgs/{org}/repos?per_page=100&type=all`):

| org | repos | pages | workers | wall clock | throughput |
| --- | --- | --- | --- | --- | --- |
| microsoft | 8,417 | 86 | 1 | 2m 19.6s | 60 repos/s |
| microsoft | 8,417 | 87 | 8 | **17.9s** | **471 repos/s** |

No 403/429 at 8-way concurrency; every response was 200. Extrapolated to a
23,000-repo organisation: **230 pages, ~50 seconds at 8-way, 230 of the PAT's
5,000 hourly requests (4.6%).**

**REST vs GraphQL** (first 5 pages of each):

| org | transport | pages | wall clock | per page | rate cost |
| --- | --- | --- | --- | --- | --- |
| google | REST | 5 | 7.6s | 1.52s | 5 requests |
| google | GraphQL | 5 | 32.6s | 6.53s | 5 points |
| microsoft | REST | 5 | 8.1s | 1.62s | 5 requests |
| microsoft | GraphQL | 5 | 24.3s | 4.86s | 5 points |

GraphQL is 3–4× slower per page for the same 100 repos, even asking for exactly
the fields discovery needs. Its usual advantage — avoiding an N+1 for the
default branch — does not apply, because REST's list payload already carries
`default_branch`. **Use REST.**

**The search API — this is the finding that decides the question:**

```
SEARCH q=tackle  status=200 hits=31 returned=20 elapsed=1.255s limit=30 remaining=29
SEARCH q=ui      status=200 hits=3  returned=3  elapsed=597ms  limit=30 remaining=28
SEARCH q=a       status=200 hits=27 returned=20 elapsed=1.094s limit=30 remaining=27
```

- **30 requests per minute, for the whole token.** A typeahead firing on
  keystrokes exhausts that in seconds, and it is shared with every other user of
  that credential.
- **~0.6–1.3s per query** before the hub adds its own hop. Not sub-second.
- **Hard cap of 1,000 results**, verified:

```
CAP page=1  (offset 0)    status=200
CAP page=10 (offset 900)  status=200
CAP page=11 (offset 1000) status=422 {"message":"Only the first 1000 search results are available", ...}
```

`GET /orgs/{org}/repos` has **no name-filter parameter at all**. So for a
23,000-repo organisation there is no GitHub call that answers "repositories in
this org matching `pay`" — search can't see past the first 1,000 matches and
can't be called often enough, and the list endpoint can't filter.

**Conditional requests are free.** Verified against the core limit:

```
ETAG 1st  status=200 etag=W/"3a79..." remaining=4751
ETAG 2nd  status=304 (conditional)    remaining=4751   <- unchanged
ETAG 3rd  status=200 (unconditional)  remaining=4750
```

A 304 does not decrement `X-RateLimit-Remaining`. Useful, but ETags are
per-page and page boundaries shift whenever a repo is added or removed, so
revalidating a 230-page listing is only cheap when nothing changed at all.

**Branches** are a second, smaller problem: `GET /repos/{o}/{r}/branches` pages
at 100 (`konveyor/tackle2-hub` → 74, one page, 337ms; `microsoft/vscode` → 100+,
818ms, more pages). Per-repo and on demand, this one is fine live.

**Documented limits** (docs.github.com, rate-limits-for-the-rest-api): PAT
5,000/hr; GitHub App installation 5,000/hr base, +50/hr per repository above 20
and +50/hr per user above 20, **capped at 12,500/hr**; Enterprise Cloud
installations 15,000/hr. Secondary limits: no more than 100 concurrent requests,
900 points/minute, 90s CPU per 60s wall. At 8 workers the measured crawl ran at
roughly 290 requests/minute — comfortably inside all three.

### 2.2 Does it belong in the hub?

**The process is right.** The issue implies the hub would be growing an entirely
new capability. It would not, quite:
[`internal/tracker/jira.go`](internal/tracker/jira.go) is already a synchronous
third-party REST client living in the hub process. It decrypts its credential
exactly the way the issue proposes
(`_ = secret.Decrypt(r.tracker.Identity)`, line 38), builds an authenticated
`http.Client` per call (`JiraConnector.client`, line 229) and is driven by a
background `Manager` loop
([`internal/tracker/manager.go:35`](internal/tracker/manager.go)) that polls,
caches connection state on the model and retries. A `platform` connector of the
same shape is a known quantity in this codebase.

It also helps that **the hub runs single-replica** — the operator's defaults are
`hub_deployment_replicas: "1"` with `hub_deployment_strategy: "Recreate"` — so
an in-process cache is coherent by construction. No distributed-cache problem to
solve.

**The request path is wrong.** Given §2.1, a `GET /platforms/{id}/repositories?search=`
that calls GitHub cannot be built as a pass-through. It would have to:

1. hold a full copy of the org's repositories (230 pages, ~50s to build),
2. serve `search=` from that copy, because GitHub cannot,
3. decide when the copy is stale and re-crawl, and
4. do all of that per platform, per identity, bounded in memory, while not
   blocking the gin worker pool.

At which point the "synchronous endpoint" is a cache with an HTTP veneer, and
the interesting design question is where the cache lives — not whether the
endpoint is synchronous.

### 2.3 Recommendation

**Defer option 2, and if it is funded, build it as a hub table filled by the
addon — not as a GitHub client in the request path.**

Two reasons to defer:

- **Option 1 largely eliminates the need.** After a discovery run the
  repositories *are* applications in the hub. The application form's repository
  typeahead can then search the hub's own inventory with the existing filter
  package ([`internal/api/filter`](internal/api/filter)), which is a plain
  indexed query. Option 2 only earns its keep for "add one application for a
  repository that was deliberately not imported" — a much narrower case than the
  issue's framing suggests.
- **It is the expensive half.** A live GitHub client in the request path brings
  cache invalidation, memory bounds, per-identity partitioning, secondary
  rate-limit backoff and request-scoped timeouts into the hub process.

If it is built anyway, the cheap shape is:

- a `PlatformRepository` table (`platform_id, owner, name, url, default_branch,
  visibility, archived, topics, last_seen`) with an index on
  `(platform_id, name)`;
- a `platform-refresh` task, same addon and same GitHub client as discovery,
  that re-crawls and upserts — measured at ~50s and 230 requests for 23,000
  repos, so a daily or on-demand refresh is nothing;
- `GET /platforms/{id}/repositories?search=` reading **only the hub DB**, via
  `BaseHandler` + `Page` + the filter package;
- `GET /platforms/{id}/repositories/{owner}/{repo}/branches` live against
  GitHub, since it is one request per repo on demand (measured 337–818ms) and
  branch lists go stale fast.

That keeps exactly one GitHub client in the codebase, in the addon, and makes
the browse endpoint an ordinary paginated query.

### 2.4 If it does go in the request path: credential and proxy

Both mechanics are already settled by existing code, and one of them is messier
than the issue assumes.

**Credential** — follow `internal/scm/mirror.go:137`, but *not* its scoping:

```go
md := &model.Identity{}
db = db.Where("kind", "source"); db = db.Where("default", true)
err = db.First(md).Error
...
err = secret.Decrypt(md)
```

For a platform, resolve `platform.IdentityID` directly (the addon already does
this via `addon.Identity.Decrypted().Get(...)`) and fall back to the default
`source` identity only if unset. Decryption itself is one `secret.Decrypt` call;
`internal/secret` handles the AES-GCM/CFB details.

**Proxy — there are two mechanisms and they do not agree.**

- `model.Proxy` ([`internal/migration/v24/model/core.go:187`](internal/migration/v24/model/core.go)),
  the thing the UI's Proxies page configures, is consumed by **nothing except
  git and subversion**. `internal/scm/pkg.go:82 proxyMap()` loads the rows and
  `shared/scm/git.go:444 (*Git).proxy()` writes `proxy = …` into a git config
  file; subversion does the same into a servers file. There is no
  `http.Transport` anywhere built from `model.Proxy`.
- The hub pod separately receives `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` from
  the operator (`roles/tackle/templates/deployment-hub.yml.j2`), which Go's
  `http.DefaultTransport` honours automatically via `ProxyFromEnvironment`.
- So `internal/tracker/jira.go:229`, which clones `http.DefaultTransport`,
  honours the pod env proxy and silently ignores `model.Proxy` entirely.

A new hub-side GitHub client inherits that split. It gets env-proxying for free;
honouring `model.Proxy` as well is roughly 40 lines (build `Transport.Proxy`
from the `http`/`https` rows, turn `Excluded` into a `NO_PROXY`-style matcher,
turn `Proxy.Identity` into a `Proxy-Authorization` header). Whether that is
wanted is a product decision, but it should be a deliberate one rather than
inherited by accident from whichever transport gets copied.

**Incidental bug found while checking this.** `proxyMap()`
([`internal/scm/pkg.go:82`](internal/scm/pkg.go)) copies
`p.Identity.Password` and `p.Identity.Key` straight out of the model with no
`secret.Decrypt` call, while `mirror.go:151` does decrypt its identity. Identity
secrets are encrypted at rest (`secret.Encrypt` at
[`internal/api/identity.go:208`](internal/api/identity.go) and `:295`; the model
tags are `secret:""`) and there is no decrypt-on-read hook — the only gorm
serializer registered is the plain JSON one in
[`internal/model/serializer.go`](internal/model/serializer.go). So an
authenticated proxy configured for git or svn is being handed ciphertext as its
password. Unrelated to this spike; worth its own issue.

---

## 3. PAT versus GitHub App

### 3.1 `Identity` fits a GitHub App better than the issue assumes

The issue says a GitHub App "may not fit `Identity` at all". It does fit:

```go
// internal/migration/v24/model/core.go:199
type Identity struct {
	Model
	Kind        string `gorm:"index;not null"`
	Name        string `gorm:"index;unique;not null"`
	Default     bool
	Description string
	User        string
	Password    string `secret:""`
	Key         string `secret:""`
	Settings    string `secret:""`
	CaCert      string
	...
}
```

A GitHub App needs three things: an App ID, an Installation ID, and a PEM
private key. `Key` already holds PEM private keys (SSH), and `Settings` is a
free-form **encrypted** string — today it carries a Maven `settings.xml`, so a
small JSON blob `{"appId": …, "installationId": …}` is well within what it is
for. No migration, no new columns.

What does *not* exist is the auth machinery: a GitHub App must mint a
10-minute RS256 JWT from the PEM, exchange it at
`POST /app/installations/{id}/access_tokens` for a ~1-hour installation token,
and refresh that token. Nothing in the hub does anything like this today — every
credential it handles is static.

### 3.2 Rate limit is not the reason to want one

Measured: a full 23,000-repo enumeration is ~230 requests. Against a PAT's
5,000/hr that is 4.6% of the budget for a complete crawl of the largest estate
anyone has described. A GitHub App's 12,500/hr cap (the scaling formula pins a
23,000-repo org to the cap) is 2.5× a budget that is already 20× larger than the
workload. **The App buys nothing for discovery throughput.**

Where an App does win is operational: it installs org-wide rather than belonging
to a person, survives that person leaving, scopes to selected repositories, and
shows up in audit logs as itself. Those are good arguments — they are just not
the arguments the issue makes.

### 3.3 Recommendation

**Ship PAT-only.** A fine-grained PAT with `metadata:read` and `contents:read`,
stored as an existing `source`-kind identity. Zero new model surface, zero new
hub code, works today with `platform.identity`.

**Treat the App as a separate, later piece**, and when it comes:

- give it its own identity `kind` (`github-app`) rather than overloading
  `source`, so the UI can render the right form and `mirror.go`'s
  `kind=source` lookups are unaffected;
- App ID and Installation ID in `Settings`, PEM in `Key`;
- put the JWT→installation-token exchange and its cache in whichever component
  owns the GitHub client (per §2.3, the addon).

**The hidden cost is in the clone path, not the discovery path.** If App auth is
also meant to serve clones, `shared/scm/git.go` needs the installation token as
its password — and installation tokens expire hourly, which
`(*Mirror).identity()` ([`internal/scm/mirror.go:137`](internal/scm/mirror.go))
is not built for: it reads a static credential once per mirror update. Making
clones work with App auth means a token-refresh hook in the SCM layer. That is
the real estimate driver, and it is worth deciding up front whether App auth is
for discovery only or for everything.

---

## 4. Provider scope: one kind per provider

**Recommendation: one kind per provider — `github`, `gitlab`, `bitbucket` — and
no separate kind for GHE or self-hosted GitLab.**

**Why not one `git-forge` kind.** `jsd.Manager.Key`
([`internal/jsd/manager.go:104`](internal/jsd/manager.go)) keys schemas
`domain.variant.subject`, and the variant is the platform kind. So the kind is
not just a dispatch label — one kind means exactly one filter schema, one
coordinates schema and one UI filter component. A single `git-forge` kind forces
a filter schema that is the union of three vocabularies, expressed with
`oneOf`/`if-then` so that `organization` applies when provider is GitHub and
`workspace` when it is Bitbucket. That complexity then has to be rendered by the
UI's generic `SchemaDefinedField`, the least flexible layer in the chain. It
also makes `Platform.Kind` stop meaning what it means for `cloudfoundry`.

**Why not a `ghe` kind.** GitHub Enterprise Server speaks the same API at a
different base URL. That is precisely what `Platform.URL` is for — the Cloud
Foundry provider already takes `URL` from the platform record. One `github` kind
covers github.com and GHE; one `gitlab` kind covers gitlab.com and self-hosted.

**On "workspace" being Bitbucket's word.** Keep each provider's own vocabulary
in its own filter schema: `organizations` for GitHub (which covers user accounts
too — `GET /users/{user}/repos` is the same shape), `groups` for GitLab,
`workspaces` for Bitbucket. Users search for the word their provider uses. A
neutral term like "scope" would be wrong in all three.

**Share implementation, not the kind.** In the addon, one `gitforge` package
with an internal `Forge` interface (`ListRepositories(filter) ([]Repo, error)`,
`ListBranches(repo)`) and three thin adapters, then three one-line `Provider`
registrations in `selectProvider`. Pagination, retry, backoff, glob matching and
the repo→`api.Application` mapping are written once. The second provider should
cost a fraction of the first.

---

## 5. Where the issue's assumptions need correcting

Two were to be confirmed from the hub side; both hold. Four others turned up.

**✅ "The hub has no git-provider API client."** Correct.
[`shared/scm/`](shared/scm) is git and subversion over their own protocols —
`SCM` is `Fetch/Update/Branch/Commit/Push/Head/Clean`
([`shared/scm/pkg.go:28`](shared/scm/pkg.go)). No `go-github`, `go-gitlab`,
`api.github.com` or `oauth2` anywhere outside a test file.
*Worth adding:* the hub does already run a synchronous third-party REST client —
the Jira tracker (§2.2) — so the pattern is not foreign, even if the GitHub
client is new.

**✅ "`internal/scm/mirror.go` resolves the source identity as `kind=source AND
default=true` globally."** Correct, verbatim —
[`internal/scm/mirror.go:137-163`](internal/scm/mirror.go). It never consults
the application's own identity, matching what
[`internal/trigger/identity.go`](internal/trigger/identity.go) documents as the
general rule.

**❌ `SourcePlatform` does not have a `coordinates` field.** The issue describes
it as "(`kind`, `name`, `url`, `identity`, `coordinates`)". The model
([`internal/migration/v24/model/platform.go:15`](internal/migration/v24/model/platform.go))
is `Name, Kind, URL, IdentityID, Identity, Applications, Tasks` — no
`Coordinates`, and `api.Platform` has no such field either. `Coordinates` is on
**`Application`** ([`internal/migration/v24/model/application.go:20`](internal/migration/v24/model/application.go)),
which is what the `platform/{kind}/coordinates` schema describes.

This matters: "Platform URL plus organisation in `coordinates`" from the issue's
option 1 has nowhere to put the organisation. Two ways out — add `Coordinates`
to `Platform` (a v25 migration, an API field and a UI form field), or **put the
organisation in the discovery filter instead and leave `Platform.URL` as the API
base**. The second needs no migration and is what §1.2 assumes. Worth settling
explicitly, because it changes whether a platform record is bound to one
organisation or can discover across several.

**⚠️ JSON Schema validation on the API is currently a no-op.**
`BaseHandler.Validate` ([`internal/api/base.go:251`](internal/api/base.go)) runs
`jsd.New(rtx.Client).Validate(r)` on every bound resource, and
`Validator.fields` ([`internal/api/jsd/validation.go:41`](internal/api/jsd/validation.go))
collects fields by type switch on `Document` / `*Document` — meaning
`internal/api/jsd.Document`. But every resource uses `*api.Document` from
[`shared/api/jsd.go:4`](shared/api/jsd.go): `resource.Application` is
`type Application api.Application`
([`internal/api/resource/application.go:19`](internal/api/resource/application.go)),
so its `Coordinates` is `*api.Document`. Those are distinct named types in
different packages, so the type switch never matches and **no document is ever
validated against its schema**. Do not design on the assumption that the hub
will reject malformed `coordinates` — whatever the addon posts is stored.
(Silver lining: it also means `Validate` makes no Kubernetes call per request,
which would otherwise be a real cost at 23,000 creates, since
`k8s.NewClient()` ([`internal/k8s/pkg.go:27`](internal/k8s/pkg.go)) builds an
*uncached* `client.New`, not a cached one.)

**⚠️ Schema CRs are duplicated across two repositories.** §1.1. Real clusters
get them from the operator template; the hub's embedded copy only feeds the
disconnected simulator. Both need updating for a new kind.

**⚠️ `model.Proxy` is honoured by git and subversion only.** §2.4. Anything new
making HTTP calls from the hub gets the pod's `HTTP_PROXY` env for free and
ignores the Proxies page unless it is explicitly wired. Plus the ciphertext
proxy-password bug noted there.

---

## 6. Sequencing and rough sizes

Sizes are one engineer, elapsed, including review and tests. The two deferred
items are listed for completeness, not recommended now.

| # | Piece | Repo(s) | Size |
| --- | --- | --- | --- |
| 1 | Settle kind name, filter schema fields, and where the organisation lives (filter vs. a new `Platform.Coordinates`) | — | **XS** · 1–2 days, mostly agreement |
| 2 | `github-filter` + `github-coordinates` Schema CRs | operator + hub | **XS** · ~1 day |
| 3 | `github.Provider`: REST client, pagination, 8-way concurrency, filtering, glob names, repo→application mapping, secondary-rate-limit backoff, tests | addon-platform | **M** · 1.5–2.5 weeks |
| 4 | Import-path fixes: skip `Fetch` for manifest-less kinds; `owner/repo` naming; summarise collisions instead of per-app warnings | addon-platform | **S** · 3–5 days |
| 5 | `KIND_MAP` entry, `filter-input-github.tsx`, i18n | tackle2-ui | **S** · 3–5 days |
| 6 | *If measurement shows it is needed:* `POST /batch/applications` | hub | **S** · ~1 week |
| — | **Subtotal for the 23,000-repository case (1–5)** | 4 repos | **≈ 4–6 weeks** |
| 7 | *Deferred.* Browse endpoints: `PlatformRepository` table + migration, `platform-refresh` task, cached read endpoint, live branches endpoint | hub + addon + UI | **M–L** · 3–4 weeks |
| 8 | *Deferred.* GitHub App auth: `github-app` identity kind, JWT→installation-token exchange and cache, **plus token refresh in the SCM clone path** | hub + addon + UI | **M–L** · 3–4 weeks |

Steps 1 and 2 are prerequisites for everything. 3 is the critical path; 4 and 5
can run alongside it once the filter schema in 1 is agreed. 6 should be measured
before being scheduled — 23,000 sequential creates may simply be acceptable for
a once-per-estate import.

Answering the issue's last question directly: **none of this needs to wait for
Cloud Foundry to settle.** The only shared code is `selectProvider` and the
`Provider` interface, both of which a GitHub provider extends rather than
changes.

---

## Appendix: reproducing the measurements

The throwaway client lives outside the repository (`/tmp/gh-spike/`), ~300 lines
of `net/http` with no dependencies. It is deliberately not committed — it proves
the numbers, it is not the start of an implementation.

```
GH_TOKEN=$(gh auth token) go run . all          # REST vs GraphQL, search, branches
GH_TOKEN=$(gh auth token) go run . extra        # ETag/304 cost, search 1000-result cap
GH_TOKEN=$(gh auth token) go run . crawl microsoft 8   # concurrent full-org enumeration
```

Headline results are inline in §2.1. Rate-limit state at the start of the run,
for context:

```
core   {limit:5000, remaining:4852}
search {limit:30,   remaining:30}
graphql{limit:5000, remaining:4885}
```
