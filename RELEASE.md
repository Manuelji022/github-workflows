# Release procedure

The reusable workflows are consumed by Git references. Publish a compatibility
tag only after the implementation and its caller examples have been reviewed.

## First `v1` tag

From a clean clone, verify that the intended commit is the current default
branch, then create and push an annotated tag:

```bash
git fetch origin main
git switch --detach origin/main
git tag -a v1 -m "Release reusable workflows v1"
git push origin refs/tags/v1
```

Confirm the tag points to the reviewed commit before enabling it in consumers.
For reproducible pinning, consumers may continue to use the full commit SHA
instead of `@v1`.

## Updating the compatibility line

Treat every `v1` move as a reviewed release. Verify that the change remains
backward compatible, create the tag on the reviewed commit, and update the
remote tag only through an explicitly approved release operation. Never move
the tag to an unreviewed branch or working-tree commit.
