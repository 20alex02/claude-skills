---
name: feature
description: Orchestrate the full multi-model feature pipeline - research context (Haiku), propose a plan for approval (Opus), implement it (Sonnet), then get an independent review (Opus). Use when the user wants a new feature, enhancement, or change to existing functionality built end-to-end via /feature, or explicitly asks to run this pipeline. Add the -test flag to also have tests written or updated. For bug reports use /bugfix instead.
---

Run this task through four stages, in order, using the pinned-model subagents from this plugin (`context-gatherer`, `planner`, `implementer`, `reviewer`; when installed as a plugin they may be listed with a `dev-pipeline:` prefix, e.g. `dev-pipeline:context-gatherer` - use whichever name is available). Do not skip a stage or merge them into one step - the point of this pipeline is that each stage runs on the model suited to it. Subagents cannot talk to the user or to each other: you relay everything and carry the context between them.

Request: $ARGUMENTS

If the request contains the `-test` flag, remove it from the task text and treat tests as in scope (see "Tests" below). Otherwise tests are out of scope.

If the request has no clear goal or no idea where it applies, ask one short clarifying question before starting.

## Working state you maintain

Subagents start with an empty context and see only what you put in their prompt, so keep these in your own notes and pass each stage what it needs:
- **Task**: the request (without the flag) plus any clarifying answers from the user.
- **Test flag**: whether `-test` was given.
- **Context brief**: output of the context-gatherer.
- **Approved plan**: the final plan after any revisions.

## Tests

- **Without `-test`**: no test work in the plan, and the implementer must not write or modify tests. It still builds and runs the existing tests as a regression check and reports the result, including failures.
- **With `-test`**: the plan includes a test section. The implementer adds tests for new functionality, or updates the existing tests for the changed behavior when the task only modifies existing functionality, then builds and runs the tests.

Tell the planner, implementer, and reviewer which mode applies so the reviewer does not flag missing tests the user did not ask for.

## Stages

1. **Research** - invoke `context-gatherer` with the task. Tell it this is a feature: find the closest existing feature to model on, the conventions to follow, the files and config that will be touched, and the existing tests for that area. It must not propose a design. Save its output as the context brief.

2. **Plan** - invoke `planner` with the task, context brief, and test flag. It returns a concrete implementation plan. If it lists open questions, relay them to the user, add the answers to the task, and re-run this stage.
   Present the plan to the user and wait for explicit approval. If they ask for changes, re-run `planner` with the task, context brief, previous plan, and the user's feedback, then present the new plan; repeat until approved. Do not edit the plan yourself and do not implement anything yet.

3. **Implement** - once the user approves the plan, invoke `implementer` with the approved plan and the test mode from "Tests". It makes the code changes, builds, runs the existing tests, and reports what it changed file by file plus the build/test output. It must not commit.

4. **Review** - invoke `reviewer` with the approved plan, the test mode, and the implementer's report. Tell it to inspect the actual changes itself (`git diff`, or the files listed in the report if the project is not a git repo) rather than trusting the report. It checks correctness, security, and convention issues and reports findings ranked by severity.
   Present the reviewer's findings to the user. If there are real findings, ask whether to address them now or leave them for later - don't silently fix or silently ignore them.

Keep your own narration between stages brief: one line saying which stage is starting and why, not a restatement of each subagent's full output. The plan, open questions, and reviewer findings are the exceptions - the user needs to read those.
