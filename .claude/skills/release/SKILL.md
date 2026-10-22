---
name: release
description: >
  Proposes the next version of this repository's reusable workflows from the
  commits since the last release, shows the exact `./release vX.Y.Z` line and
  runs it only after the user says yes. Use when the user asks to release,
  cut a tag, or move v1/v2 onto main.
disable-model-invocation: true
---

# release

Choose the next version for `./release`, then run it with the user's consent.

## Rules

1. Never run `./release` without an explicit yes to the exact line shown.
   A release is outward-facing: callers on `@vX` pick it up at once.
2. Never create or move a tag yourself. `./release` is the only way a tag
   is made here, and it refuses what it cannot verify.
3. The rule for patch, minor and major is the README's Versions section.
   Apply it; do not restate or reinterpret it.

## Steps

1. Find the last release and what came after it:

   ```bash
   last="$(git tag --list 'v[0-9]*.[0-9]*.[0-9]*' | sort -V | tail -1)"
   git log --oneline "${last}..main"
   ```

   Read the README's Versions section for the rule. If the log is empty,
   say there is nothing to release and stop.

2. Propose patch, minor or major. For each commit that drives the choice,
   one line: the commit, and why it is a fix, new compatible behaviour, or
   a break (renamed input, changed default, removed workflow). Say which
   single commit sets the level: one break makes the whole release major.

3. Show the exact line and stop:

   ```
   ./release v1.2.0
   ```

   Wait for the user's yes. A different number from the user wins; a
   question is not a yes.

4. On yes, run the line and show its output as is. If it refuses, show the
   refusal and its fix line, and stop; do not work around it.

5. On a major: remind that callers stay on the old `@vX` until each one
   moves its `uses:` line itself.
