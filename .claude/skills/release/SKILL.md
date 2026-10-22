---
name: release
description: >
  Cuts a release of this repository's reusable workflows: proposes the next
  vX.Y.Z from the commits since the last tag, then runs ./release on the
  user's yes. Use when the user asks to release, cut a tag, or move v1/v2
  onto main.
disable-model-invocation: true
---

# release

## Rules

1. Never create or move a tag yourself. `./release` is the only way a tag
   is made here.

## Steps

1. Bring the tags and main up to date, then find the last release and what
   came after it. Stop if the fetch fails.

   ```bash
   git fetch --tags origin main
   last="$(git tag --list 'v[0-9]*.[0-9]*.[0-9]*' | sort -V | tail -1)"
   git log --oneline "${last}..origin/main"
   ```

   Read the README's Versions section for the rule. If the log is empty,
   say there is nothing to release and stop.

2. Propose patch, minor or major. For each commit that drives the choice,
   one line: the commit, and why it is a fix, new compatible behaviour, or
   a break. Say which single commit sets the level.

3. Show the exact line and stop:

   ```
   ./release v1.2.0
   ```

   A release is outward-facing: callers on `@vX` pick it up at once, so
   wait for the user's yes. If the user gives a different number, show the
   new line and wait for a yes again. A question is not a yes.

4. On a yes to the line shown, run exactly that line and show its output as
   is. If it refuses, show the refusal and its fix line, and stop; do not
   work around it.

5. On a major, point at the README's major bullet.
