# Resolve a repository's default branch (and branch list) from its URL, without a provider API

> Draft issue text. Suggested repo: **konveyor/tackle2-hub** (the substantive work
> is hub-side); the form change is a small linked issue on **konveyor/tackle2-ui**.
> Related: konveyor/tackle2-ui#3704 (org discovery) — this is independent of it and
> much smaller.

## Problem

When you add an application, the branch is free text you type from memory
(`RepositoryFields` in the UI's `application-form.tsx`). Nothing checks it. A wrong
branch does not fail at the form — it fails much later, when an analysis or an agent
run tries to clone, with an error that points at the task rather than at the typo.

It is easy to get wrong because the default is not guessable:

```
konveyor/mig-demo-apps   default branch = master
microsoft/vscode         default branch = main
```

Anyone typing `main` for the first one has broken their application and will find
out an hour later.

This is a different problem from onboarding at scale (#3704) and a much cheaper one.
It affects every user who ever adds a single application, and it needs no platform
kind, no provider API client, no PAT and no rate-limit handling.

## Why this is cheap: git already answers the question

`git ls-remote` queries a remote's refs **without cloning**, over the git protocol,
for any provider. Measured against the live services:

| Command | Repo | Result | Time |
| --- | --- | --- | --- |
| `ls-remote --symref <url> HEAD` | `konveyor/mig-demo-apps` | `ref: refs/heads/master` | 0.47s |
| `ls-remote --symref <url> HEAD` | `microsoft/vscode` | `ref: refs/heads/main` | 0.54s |
| `ls-remote --symref <url> HEAD` | a **private** repo, credentialed | `ref: refs/heads/main` | 0.66s |
| `ls-remote --heads <url>` | `microsoft/vscode` | **5,903 branches** | 0.59s |

No REST API, no GitHub/GitLab/Bitbucket distinction, no rate limit, works against
self-hosted GHE and self-hosted GitLab, and needs no credential scope beyond the
read access the hub already needs in order to clone.

Two findings worth knowing before implementing:

- **`--symref` and `--heads` are mutually exclusive in practice.** `ls-remote
  --symref --heads <url>` silently drops the `ref:` line. Use `--symref <url> HEAD`
  for the default branch and a separate `--heads <url>` for the list.
- **Do not call `ls-remote --symref <url>` with no refspec.** On `microsoft/vscode`
  that returns **83,192 refs** (5,904 heads, 455 tags, the rest `refs/pull/*`) and
  takes 1.57s. Always filter.

## Why this is cheap in *this* codebase

The authentication and proxy plumbing already exists and `ls-remote` inherits all of
it for free, because it is the same `git` binary run with the same prepared `HOME`.

`(*Git).initHome()` in [`shared/scm/git.go`](../../shared/scm/git.go) builds a
throwaway `HOME` containing:

- `.gitconfig` from `writeConfig()` — the credential helper, `http.sslVerify` from
  the `git.insecure.enabled` setting, and `http.proxy` from `(*Git).proxy()`, which
  is fed by `model.Proxy` rows via `internal/scm.proxyMap()`;
- `.git-credentials` from `writeCreds()` — `https://user:password@host`;
- the SSH key from `Remote.Identity`, via `ssh.Key.Add()`.

and `(*Git).git()` runs with `HOME=r.Home` and `GIT_TERMINAL_PROMPT=0`.

So a new `ls-remote` method gets HTTP basic auth, SSH auth, the configured proxy and
the TLS-verification setting with **no new code in any of those areas**.

Verified: with a `.gitconfig`/`.git-credentials` pair built exactly the way
`writeCreds()` builds them, `ls-remote --symref` against a private repository
returns its default branch; without them it fails with
`could not read Username for 'https://github.com': terminal prompts disabled`.

**Security note in favour of this approach:** credentials go in
`.git-credentials`, never on the command line and never in the remote URL, so git's
stderr — which this feature will surface — cannot leak them.

There is also already a `(*Git).defaultBranch()`
([`shared/scm/git.go:313`](../../shared/scm/git.go)) doing
`rev-parse --abbrev-ref origin/HEAD`. It needs a local clone, so it is not reusable
here, but it confirms the concept belongs in this type.

## Proposed change

### 1. `shared/scm` — a remote ref lookup

Add to `shared/scm/pkg.go`:

```go
// Refs remote references.
type Refs struct {
	Default  string   `json:"default"`
	Branches []string `json:"branches,omitempty"`
}
```

Add to the `SCM` interface:

```go
Refs(ctx context.Context) (refs Refs, err error)
```

`Refs` is the only `SCM` method that will be called from an HTTP request path, so it
is the only one that takes a `context.Context`. That asymmetry is deliberate: an
unreachable host must not pin a gin worker. DNS failures return in ~0.1s, but a
blackholed host will hang, so the caller passes a deadline.
`command.Command.RunWith(ctx)` already uses `exec.CommandContext`, so this needs no
new plumbing — only `Run()` → `RunWith(ctx)` at the call site.

`(*Git).Refs`:

1. `initHome()`;
2. `git ls-remote --symref <url> HEAD`, parse `ref: refs/heads/<name>\tHEAD` → `Default`;
3. `git ls-remote --heads <url>`, strip the `refs/heads/` prefix → `Branches`;
4. `defer Clean()`.

`(*Subversion).Refs` returns a `NotSupported` error — Subversion's `trunk` is a path
convention, not a ref, and inventing an answer would be worse than none.

**Concurrency gotcha to handle, not discover later:** `initHome()` does
`nas.RmDir(r.Home)` before recreating it, and `Home` derives from `Base.Id()`, which
hashes `Remote.Digest()` + `Path`. Two concurrent lookups for the same URL with the
same `Path` would delete each other's `HOME` mid-flight. The handler must pass a
unique per-request temp `Path` so `Id()` differs, and `Clean()` in a defer.

### 2. `internal/api` — the endpoint

Construct the SCM via `internal/scm.New(db, tempDir, remote)` rather than building
`scm.Git{}` directly — `New()` is what reads the `git.insecure.enabled` setting and
attaches the proxy map ([`internal/scm/pkg.go`](../../internal/scm/pkg.go)).

Routes in `shared/api/pkg.go`:

```go
// Routes - Repositories
const (
	RepositoriesRoute    = "/repositories"
	RepositoryRefsRoute  = RepositoriesRoute + "/refs"
)
```

`POST /repositories/refs`:

```jsonc
// request
{ "url": "https://github.com/konveyor/mig-demo-apps", "identity": { "id": 3 } }

// 200
{ "default": "master", "branches": ["master", "gh-pages", ...] }
```

**POST, not GET, is deliberate** — the repository URL goes in a body rather than a
query string (it can carry organisation names, and the project's own guidance is to
keep that out of URLs), and the call makes an outbound network request. There is
precedent for a non-CRUD action route in `CopyRoute = ReviewsRoute + "/copy"`.

Identity resolution follows the existing rule: use `identity.id` when supplied,
otherwise fall back to the default `kind=source` identity the way
[`(*Mirror).identity()`](../../internal/scm/mirror.go) does, then `secret.Decrypt`.

Map failures to a small set of outcomes rather than forwarding raw git stderr:
auth failure → `401`, host unreachable or timeout → `502`, repo not found → `404`,
Subversion → `204` with no body so the UI simply leaves the field alone.

### 3. RBAC

`Required("repositories")` self-registers the resource
([`internal/api/auth.go`](../../internal/api/auth.go) `Required()` calls
`auth.Domain().Register`), so only the seed grant is needed. In
`internal/auth/seed/roles.yaml`, add to **`architect`** (the role that already has
`applications: post`):

```yaml
    - name: repositories
      verbs:
        - post
```

`admin` is covered by its `'*'` wildcard. `migrator` and `project-manager` have
`applications: get` only and do not need it.

### 4. UI (separate, small)

On blur of the repository URL field in the application form: POST, prefill the
branch with `default` when the user has not typed one, and offer `branches` as a
typeahead. If the call fails, leave the field as free text — this must degrade to
today's behaviour, never block the form.

## Scope

**In scope:** default branch + branch list for git remotes, from a URL the user has
already entered.

**Not in scope:** discovering repositories (that is konveyor/tackle2-ui#3704),
any provider REST client, PATs or GitHub Apps, and the root-path/tree picker.

## Why this is worth doing before #3704

It fixes the failure that actually costs people time, it helps every user rather
than only the large-estate case, and it is days rather than weeks.

It also quietly delivers the branches half of #3704's option 2 — a branch typeahead
— **provider-agnostically and with no rate limit**, which the GitHub REST approach
cannot claim: `/repos/{o}/{r}/branches` pages at 100 and only works for GitHub,
whereas `ls-remote --heads` returned all 5,903 branches of `microsoft/vscode` in
0.59s and works against anything that speaks git.

## Reproducing the measurements

```bash
git ls-remote --symref https://github.com/konveyor/mig-demo-apps HEAD
git ls-remote --heads  https://github.com/microsoft/vscode | wc -l
```

For the private-repo check, build a throwaway `HOME` the way `writeCreds()` does
(a `.gitconfig` with `credential.helper = store --file=...` plus a
`.git-credentials` line of `https://user:token@github.com`) and run the same command
with `HOME` pointed at it and `GIT_TERMINAL_PROMPT=0`.
