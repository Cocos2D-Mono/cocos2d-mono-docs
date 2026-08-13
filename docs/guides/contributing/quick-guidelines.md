---
sidebar_position: 2
---

# Quick Guidelines

Here are a few simple rules and suggestions to remember when contributing to Cocos2D-Mono.

- :bangbang: **NEVER** commit code that you didn't personally write.
- :bangbang: **NEVER** use decompiler tools to steal code and submit it as your own work.
- :bangbang: **NEVER** decompile XNA assemblies and steal Microsoft's copyrighted code.
- **PLEASE** try to keep your PRs focused on a single topic and of a reasonable size or you may be asked to break it up.
- **PLEASE** be sure to write simple and descriptive commit messages.
- **DO** branch from `dev` and target your PR at `dev` — see [Getting Involved](/docs/guides/contributing/getting-involved.md).
- **DO NOT** surprise us with new APIs or big new features. Open an issue to discuss your ideas first.
- **DO NOT** reorder type members as it makes it difficult to compare code changes in a PR.
- **DO** try to follow our [coding style](/docs/guides/contributing/code-guidelines.md) for new code.
- **DO** give priority to the existing style of the file you're changing.
- **DO** try to add to the [tests](https://github.com/Cocos2D-Mono/cocos2d-mono/tree/master/Tests) when adding new features or fixing bugs.
- **DO NOT** send PRs for code style changes or make code changes just for the sake of style.
- **DO** build without new warnings. Incremental builds can hide warnings from files you didn't touch — confirm with `--no-incremental`.
- **DO** actually run what you changed. A clean build and a launch that doesn't crash prove only that the process started; they cannot catch a mis-sized hitbox, a collision filter applied to one side only, or an input path that silently stopped firing. If you couldn't verify visually, say so in the PR instead of calling it verified.
- **DO** open a separate issue and PR when you find an unrelated bug along the way, rather than folding it into the change you're working on — especially if that change is a mechanical one.
- **DO** update the matching [samples](https://github.com/Cocos2D-Mono/Cocos2D-Mono.Samples) checkpoint when you change code that a tutorial teaches, and vice versa. The docs quote that code verbatim.
- **PLEASE** keep a civil and respectful tone when discussing and reviewing contributions.
- **PLEASE** tell others about Cocos2D-Mono and your contributions via social media.
