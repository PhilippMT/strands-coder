# GitLab adaptation research

This document captures the refined path for running `strands-coder` against a self-hosted GitLab instance.

## Current implementation fit

- `strands-coder` is a Python CLI package with a GitHub Action wrapper.
- The agent runner already supports container execution, environment-driven prompts, Strands tools, MCP servers,
  S3-backed sessions, and OpenTelemetry/Langfuse.
- GitLab support should therefore reuse the CLI entry point instead of creating a separate runner.
- GitLab payloads are now accepted through `GITLAB_CONTEXT`, `GITLAB_EVENT_PAYLOAD`, or `STRANDS_EVENT_PAYLOAD`.
- Native GitLab CI variables are used when no webhook payload is supplied.

## Dependency research

Latest compatible package versions verified on PyPI for Python 3.10+:

| Package | Selected version floor | Reason |
| --- | ---: | --- |
| `strands-agents[all]` | `1.37.0` | Latest Strands Python SDK; keeps all Strands extras available. |
| `strands-agents-tools` | `0.5.1` | Latest compatible community tools package. |
| `requests` | `2.33.1` | Latest compatible HTTP client release. |
| `colorama` | `0.4.6` | Latest stable release. |
| Dev tooling | latest compatible floors | Updated hatch, mypy, pytest, ruff, stubs, and xdist ranges. |

Security advisory lookup found no known GitHub Advisory Database vulnerabilities for the selected versions.

## Research tree

```text
Goal: adapt strands-coder to self-hosted GitLab
├── Execution environment
│   ├── GitLab CI Docker executor
│   │   ├── Best for direct repository work
│   │   ├── Uses project/group/instance runners
│   │   ├── Needs protected runners and protected variables for trusted branches
│   │   └── Recommended default path
│   ├── GitLab CI shell/Kubernetes runners
│   │   ├── Shell runner has broader host access and higher blast radius
│   │   └── Kubernetes runner is useful for strong per-job isolation
│   └── AWS Bedrock AgentCore Runtime
│       ├── Best for long-running or externally invoked agent service
│       ├── Provides session isolation and scale-to-demand runtime
│       └── Needs HTTP adapter around the CLI/runner contract
├── Trigger model
│   ├── Merge request pipeline
│   │   ├── Use `CI_PIPELINE_SOURCE == "merge_request_event"`
│   │   └── Good for commits and MR lifecycle events
│   ├── GitLab note/comment webhook
│   │   ├── GitLab sends `Note Hook` for comments on merge requests, issues, commits, and snippets
│   │   └── Best for "agent, please ..." comments
│   ├── Pipeline trigger API
│   │   ├── Accepts trigger token or CI job token
│   │   ├── Can receive variables/inputs
│   │   └── Best for Jira or other external systems
│   └── Generic webhook relay
│       ├── Validates source signatures
│       ├── Normalizes payload into `STRANDS_EVENT_PAYLOAD`
│       └── Starts GitLab CI or invokes AgentCore
├── Context acquisition
│   ├── CI variables for project, pipeline, branches, and MR metadata
│   ├── GitLab webhook payload for exact user action
│   ├── GitLab REST API for MR note history
│   └── Optional Jira payload body for external business context
└── Security controls
    ├── Run untrusted fork/MR code without secrets
    ├── Use protected variables/runners for write tokens
    ├── Prefer project access tokens with minimum scopes
    ├── Avoid exposing trigger tokens in comments or logs
    └── Restrict webhook relays by secret token, allow-list, and replay protection
```

## Decision tree

```text
Need the agent to modify the repository?
├── Yes
│   ├── Is the event from trusted code/branch?
│   │   ├── Yes → Run in GitLab CI Docker executor with protected write token.
│   │   └── No → Run read-only analysis, post no write actions, require maintainer approval.
│   └── Are jobs longer than runner limits or event-driven outside GitLab?
│       ├── Yes → Use webhook relay → AgentCore runtime → GitLab API.
│       └── No → Keep GitLab CI Docker executor.
└── No
    ├── Need fast response to comments/webhooks?
    │   ├── Yes → Webhook relay or AgentCore.
    │   └── No → Scheduled or manually triggered GitLab CI job.
    └── Need access to local checkout?
        ├── Yes → GitLab CI.
        └── No → AgentCore service is acceptable.
```

## Recommended GitLab CI path

Use GitLab CI Docker executor as the first implementation because it matches the current CLI model and naturally provides
a repository checkout.

```yaml
stages:
  - agent

strands_coder:
  stage: agent
  image: python:3.13-slim
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_PIPELINE_SOURCE == "trigger"'
    - if: '$CI_PIPELINE_SOURCE == "pipeline"'
    - when: manual
  variables:
    STRANDS_PROVIDER: bedrock
    STRANDS_PROMPT: "React to the GitLab event and help with the requested repository task."
  before_script:
    - python -m pip install --upgrade pip
    - pip install strands-coder
  script:
    - strands-coder
```

For self-hosted GitLab, configure:

- `GITLAB_TOKEN` or `CI_JOB_TOKEN` for GitLab API access.
- `STRANDS_EVENT_PAYLOAD` for external payloads, or `GITLAB_EVENT_PAYLOAD` for GitLab webhook payloads.
- `S3_SESSION_BUCKET` when sessions should persist across pipelines.
- `OTEL_EXPORTER_OTLP_ENDPOINT` or Langfuse variables for tracing.

## External triggers

### Merge request comments

Preferred flow:

1. Configure a GitLab project or group webhook with note/comment events.
2. Send the webhook to a small internal relay.
3. Validate the GitLab webhook secret token.
4. Trigger a GitLab pipeline with `STRANDS_EVENT_PAYLOAD` or `GITLAB_EVENT_PAYLOAD` containing the note payload.
5. `strands-coder` extracts the note body and MR metadata, enriches MR note history when API credentials are present,
   and runs the agent prompt.

### Jira webhooks

Preferred flow:

1. Jira calls the same internal relay.
2. The relay validates Jira authenticity and maps issue fields to `STRANDS_EVENT_PAYLOAD`.
3. The relay triggers GitLab CI with a restricted pipeline trigger token or invokes AgentCore.
4. `strands-coder` treats the payload as external trigger context and uses summary, description, comment, or body fields
   as the user message.

## AgentCore alternative

Choose AgentCore when:

- The agent must react without waiting for GitLab runner capacity.
- The same agent endpoint should serve GitLab, Jira, and other systems.
- Session isolation, horizontal scaling, or long-running sessions are more important than direct CI checkout access.

AgentCore needs an HTTP adapter that exposes a `/ping` health endpoint and an invocation endpoint, then calls the existing
runner logic with normalized environment variables. For repository write operations, the runtime still needs a GitLab
access token and should clone/fetch only the specific target repository and ref.

## Ten refinement passes applied

1. Confirmed the current package is GitHub-oriented but CLI-first.
2. Verified latest Strands SDK and tool package versions.
3. Updated runtime dependency floors and retained safe upper bounds.
4. Updated dev tooling floors to latest compatible versions.
5. Added GitLab payload parsing without breaking GitHub context parsing.
6. Added GitLab CI variable fallback for native merge request pipelines.
7. Added MR comment extraction and optional MR note-history enrichment.
8. Added external trigger payload handling for Jira-style webhooks.
9. Compared GitLab CI Docker execution with AgentCore runtime.
10. Refined the recommendation to GitLab CI Docker by default, AgentCore for externally hosted event service needs.
