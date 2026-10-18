# AGENTS.md

This repository is the sourcectl fork of [dghubble/gologin](https://github.com/dghubble/gologin).
[`FORK.md`](FORK.md) records how it diverges from upstream. This file is the procedure to
**sync** it: realign the fork with upstream, move go-github to its latest major version, and
update the consumer [sourcectl/sourcectl](https://github.com/sourcectl/sourcectl).

A sync always ends with a pull request against sourcectl/sourcectl. The fork is only useful
through that consumer.

## Terms

- `upstream`: the remote `https://github.com/dghubble/gologin.git`.
- `origin`: the fork, `git@github.com:sourcectl/gologin.git`.
- `main`: mirrors `upstream/main` exactly. Never commit to it.
- **fork branch**: `andygrunwald/upgrade-go-github-v<N>`, where `<N>` is the go-github major
  version it carries. It is `main` plus our changes, and the default branch of the fork.
- `<N>`: the major version of the latest [google/go-github](https://github.com/google/go-github) release.

## Sync

Each step ends on its **done** check. Move on only when it holds.

### 1. Preconditions

```sh
git status --porcelain                      # must print nothing
git remote get-url upstream || git remote add upstream https://github.com/dghubble/gologin.git
git fetch upstream && git fetch origin
```

**Done:** clean tree, both remotes fetched.

### 2. Fast-forward `main`

```sh
git checkout main
git merge --ff-only upstream/main
git push origin main
```

**Done:** `git rev-parse main upstream/main origin/main` prints the same commit three times.
If the fast-forward fails, someone committed to `main`: stop and ask the user.

### 3. Determine `<N>`

```sh
gh api repos/google/go-github/releases/latest --jq .tag_name   # e.g. v93.0.0 -> <N> = 93
grep go-github go.mod                                          # version upstream uses
```

**Done:** you know `<N>`, upstream's go-github version, and the current fork branch
(`gh repo view sourcectl/gologin --json defaultBranchRef`), called `<old-fork-branch>` below.

### 4. Create the fork branch from `main`

There is always a fork branch, even when upstream already uses go-github v`<N>`, because it
carries `FORK.md`, `AGENTS.md` and `CLAUDE.md`, which `main` does not have.

```sh
git checkout -b andygrunwald/upgrade-go-github-v<N> main
```

If that branch already exists (upstream moved, go-github did not), rebuild it instead: check
out the existing branch and `git rebase main`, keep its commits (resolve conflicts so that
upstream wins), run `make`, push with `git push --force-with-lease origin HEAD`, and continue
with step 8. The default branch and the old branch stay as they are.

**Done:** you are on the fork branch and it is based on the current `main`.

### 5. Upgrade go-github (one commit)

Skip this step if upstream already uses go-github v`<N>` or newer.

1. Replace the import path in every file: `git grep -l 'go-github/v' -- '*.go'`, change
   `go-github/v<old>` to `go-github/v<N>`.
2. `go get github.com/google/go-github/v<N>@latest && go mod tidy`
3. Read the release notes of every major version between `<old>` and `<N>`
   (`gh api repos/google/go-github/releases/tags/v<X>.0.0 --jq .body`) and adapt to every
   breaking change that touches code here. Keep the code identical to upstream wherever it
   still compiles, deprecated helpers like `github.Ptr` included. **Fidelity** to upstream beats tidiness.
4. Update the go-github entry under `## Latest` in `CHANGES.md` to the new version.
5. Run `make` (tests, vet, fmt).

Leave `examples/` alone. It pins a released gologin version.

**Done:** `make` passes, `git grep 'go-github/v<old>'` is empty, and the commit
`Upgrade github.com/google/go-github from v<old> to v<N>` contains only this change.

### 6. Carry over the fork docs (separate commits)

1. `git checkout <old-fork-branch> -- FORK.md`, then rewrite its divergence section to name the
   fork branch, the version jump (or "none" if step 5 was skipped), every file touched, and any
   behavior change. Remove everything that no longer applies. Commit.
2. `git checkout <old-fork-branch> -- AGENTS.md CLAUDE.md`. Update this file only where the
   procedure turned out to be wrong. Commit.

**Done:** every statement in FORK.md matches `git diff main...HEAD`, and
`git diff <old-fork-branch> -- AGENTS.md CLAUDE.md` shows only intended changes.

### 7. Publish the fork branch

```sh
git push -u origin andygrunwald/upgrade-go-github-v<N>
gh repo edit sourcectl/gologin --default-branch andygrunwald/upgrade-go-github-v<N>
gh pr list -R sourcectl/gologin --base <old-fork-branch>   # must be empty, else ask the user
git push origin --delete <old-fork-branch>
git branch -D <old-fork-branch>
```

**Done:** `gh repo view sourcectl/gologin --json defaultBranchRef` shows the new branch, and
`git ls-remote --heads origin` lists only `main` and the new fork branch.

### 8. Update sourcectl/sourcectl

The local checkout is `../sourcectl`. The consumer is `web-app/`. It passes its own
`*github.User` into gologin, so it **must** use the same go-github major version as the fork.

```sh
cd ../sourcectl
git status --porcelain --untracked-files=no   # must print nothing
git checkout main && git pull
git checkout -b andygrunwald/gologin-go-github-v<N>
cd web-app
go list -m -json github.com/sourcectl/gologin/v2@<fork-commit-sha> | grep '"Version"'   # sha = pushed fork branch HEAD
go mod edit -replace github.com/dghubble/gologin/v2=github.com/sourcectl/gologin/v2@<pseudo-version>
```

Then change every `go-github/v<old>` import in `web-app/` to `v<N>`, run
`go get github.com/google/go-github/v<N>@latest && go mod tidy`, and adapt to the breaking
changes from step 5.

Verify in `web-app/`: `go build ./... && go vet ./... && go test ./...`, and
`grep -rn 'go-github/v<old>' --include='*.go' --include=go.mod .` is empty.

Commit, push and open the PR with `gh pr create --base main`. The body says what moved
(fork commit, go-github version), why go-github moves together with gologin, and links FORK.md.

**Done:** the PR URL exists and `gh pr checks <url>` lists the `web-app` CI run.

## Commits

- Subject in the imperative mood, body explains why.
- One logical change per commit: the go-github upgrade, FORK.md, the agent docs, and the sourcectl
  update are separate commits.
