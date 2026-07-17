# JoyAgent Genie Agent Entry

## Load Order

Before changing this repository, read these files in order:

1. `.agents/context.md` for the current architecture, service boundaries, and source-of-truth files.
2. `.agents/constraints.md` for hard safety and compatibility boundaries.
3. `.agents/development.md` for implementation conventions and verification commands.

## Working Rules

- Follow the user's current request first. Keep changes minimal and limited to the requested scope.
- Inspect the relevant implementation and configuration before proposing a change; README examples may lag the code.
- Preserve existing behavior and cross-service contracts unless the task explicitly changes them.
- Do not overwrite unrelated work or generated local state. Check `git status --short` before and after editing.
- Never commit, push, install global dependencies, or modify machine-level configuration unless explicitly requested.
- Use the narrowest useful verification first, then widen checks according to risk. Report checks that could not run.
- Treat configuration, credentials, external services, LLM output, uploaded files, MCP endpoints, and database input as untrusted.

