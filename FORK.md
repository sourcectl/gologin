# Fork of dghubble/gologin

This repository is a fork of [dghubble/gologin](https://github.com/dghubble/gologin).

The goal is to stay as close to upstream as possible. We only carry changes that
we need before upstream ships them. Once upstream contains an equivalent change,
we drop ours and sync with upstream again.

The module path is unchanged (`github.com/dghubble/gologin/v2`).

To sync the fork with upstream, follow [`AGENTS.md`](AGENTS.md).

## Changes compared to upstream

All changes live on the branch `andygrunwald/upgrade-go-github-v93`, which is the
default branch of this fork. `main` mirrors upstream `main`.

### Upgrade `github.com/google/go-github` from v89 to v93

Upstream already upgraded go-github from v64 to v89
([4f39173](https://github.com/dghubble/gologin/commit/4f39173bc2f19f0ed985f24adf84731ec0b0b3b4),
[acc87f1](https://github.com/dghubble/gologin/commit/acc87f1acf9f8dc8b30f6787d92078fa9df655e9)).
We need v93 because consumers (sourcectl `web-app`) pass their own
`*github.User` into gologin, which only compiles when both use the same
go-github major version.

What changed:

- `github/*.go`: import `github.com/google/go-github/v93/github`.
- `go.mod` / `go.sum`: go-github v89 → v93.
- `CHANGES.md`: the upstream changelog entry now reads v64 → v93.

No behavior change: client construction and the GitHub Enterprise URL handling
are identical to upstream. Tests keep using `github.Ptr` (deprecated since v92)
to stay close to upstream.

## Removing the fork changes

Once upstream ships go-github v93 or newer, the upgrade commit is dropped during
the next sync (see [`AGENTS.md`](AGENTS.md)). The fork branch then carries only
the fork documentation.
