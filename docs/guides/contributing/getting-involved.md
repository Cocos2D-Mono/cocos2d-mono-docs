---
sidebar_position: 1
---

# Getting Involved

In order to contribute to the project you should learn and be familiar with how to [use Git](https://help.github.com/articles/set-up-git/), how to [create a fork of Cocos2D-Mono](https://help.github.com/articles/fork-a-repo/), and how to [submit a Pull Request](https://help.github.com/articles/using-pull-requests/).

## Branching

Cocos2D-Mono uses the following branches:

- `master` (or `main` in the other repositories) — **release-only.** Stable releases live here; it is never a target for feature work.
- `dev` — continued development, and the **base for everything you write**
- `release/**` — prepares and stabilizes a release before it merges to the default branch
- `feature/**`, `fix/**`, `update/**`, `docs/**`, `chore/**` — the work itself

The rule that matters: **branch from `dev`, and target your pull request at `dev`.**

```bash
git checkout dev
git pull --ff-only
git checkout -b feature/my-change
```

This applies across every Cocos2D-Mono repository — the engine, docs, samples, project
templates, and tests. Only release branches and release-coupled content go to the default
branch.

## Pull requests

Keep pull requests small and focused on a single topic; a large change split into a
sequence of focused PRs lands faster and is easier to revert if something goes wrong.
Where practical, keep mechanical changes (renames, file moves) in a separate PR from
behavior changes, so reviewers can read each one for what it is.

After you submit a PR, your changes will be reviewed and provided with any constructive
feedback to improve your submission. Once your changes are good for cocos2d-mono, your PR
will be merged.

## Where changes live

The project spans several repositories, and a change to one often has a counterpart in
another:

| Repository | Contents |
| --- | --- |
| [`cocos2d-mono`](https://github.com/Cocos2D-Mono/cocos2d-mono) | The engine, the Box2D port, and the test projects |
| [`Cocos2D-Mono.Docs`](https://github.com/Cocos2D-Mono/Cocos2D-Mono.Docs) | This documentation site |
| [`Cocos2D-Mono.Samples`](https://github.com/Cocos2D-Mono/Cocos2D-Mono.Samples) | Sample games and the tutorial sample projects |
| [`Cocos2D-Mono.ProjectTemplates`](https://github.com/Cocos2D-Mono/Cocos2D-Mono.ProjectTemplates) | `dotnet new` templates and the Visual Studio extension |
| [`Cocos2D-Mono.Tests`](https://github.com/Cocos2D-Mono/Cocos2D-Mono.Tests) | A multi-platform showcase app built against the published packages |

Notably, the tutorial pages on this site show code that readers copy into their own
projects, and that code is kept **verbatim** in sync with the checkpoints in the samples
repository. If you change one, change the other in the same round of work.

The engine repository's [`CONTRIBUTING.md`](https://github.com/Cocos2D-Mono/cocos2d-mono/blob/master/CONTRIBUTING.md)
carries the same conventions alongside the code, including the release flow.
