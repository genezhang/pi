# FORK.md — fork layout & upstream sync

This repo (`genezhang/pi`) is a fork of **`earendil-works/pi`**. The goal of this
layout is to stay *trivially re-syncable* with a fast-moving upstream.

## Governing principle: adapt via extensions, not core forks

Pi is a **self-extensible** agent: its extension API (see
[`packages/coding-agent/docs/extensions.md`](packages/coding-agent/docs/extensions.md))
lets you add tools, hooks, commands, and behavior as drop-in TypeScript loaded at
runtime — **without modifying Pi's source**. Use that surface for customizations.

> We learned the hard way with OpenCode that *intrinsic* core changes (forking and
> editing the engine itself) make catching up with upstream a constant, painful
> merge. Pi avoids this by design — so keep our customizations **out of the fork**.

Concretely:
- Customizations (e.g. the zengram memory integration) live in **their own repos**
  as Pi extensions, installed at runtime. They cause **zero fork divergence**.
- Put a change on the `work` branch **only if the extension API genuinely cannot
  express it** (a missing hook point, a core bug). When that happens, prefer
  **upstreaming it as a PR** to `earendil-works/pi` — the rebase layout below makes
  that easy, and it shrinks our long-term maintenance burden to zero.

The success metric for this fork is: `git diff upstream/main...work` stays as
small as possible.

## Branch layout

| Remote | Repo |
|---|---|
| `upstream` | `earendil-works/pi` (source of truth) |
| `origin` | `genezhang/pi` (this fork) |

| Branch | Role |
|---|---|
| `main` | **Pristine mirror** of `upstream/main`. Never commit here. Configured to *fetch from upstream, push to origin*. |
| `work` | Our integration branch — the GitHub **default branch**. Holds only changes that can't be extensions. |

## One-time setup (already applied)

```bash
git remote add upstream git@github.com:earendil-works/pi.git
git config branch.main.remote     upstream     # `git pull` on main ff's from upstream
git config branch.main.pushRemote origin       # `git push` on main updates the fork mirror
git config branch.main.merge      refs/heads/main
# GitHub default branch set to `work` (so clones/PRs land on our code, main stays a reference)
```

## Check divergence from upstream (the "what did we change?" answer)

```bash
git fetch upstream
git log  upstream/main..work     # exactly our commits on top of upstream
git diff upstream/main...work    # exactly our net changes
```

## Absorb upstream advances

```bash
git fetch upstream
git checkout main && git pull              # ff-only mirror update (pull=upstream)
git push origin main                       # refresh the fork's mirror (optional)

git checkout work && git rebase upstream/main
# resolve conflicts if any (rare while work stays small)
git push --force-with-lease origin work
```

## Rebase vs merge

**Rebase** (above) keeps `work` = "upstream + a readable patch stack" — trivial to
review and to upstream. Cost: it rewrites `work`'s history, so push with
`--force-with-lease`. Only use **merge** instead if someone else is building on
`work` and can't handle a force-push.
