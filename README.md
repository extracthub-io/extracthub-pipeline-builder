# ExtractHub pipeline builder (Agent Skill)

An [Agent Skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) that teaches an
AI agent to build [ExtractHub](https://extracthub.io) pipelines: multi-step scrapes that chain released
extractors (a list or category page, then every detail page it links to) into one dataset. The agent finds or
builds the extractors, designs and validates the chain, tells you what a run costs, creates the pipeline,
runs it small once with your OK, checks the results and sets a schedule only when you agree.

Ask your agent things like:
- "Scrape every book of this Books to Scrape category and each book's detail page into one dataset."
- "Take the product ids from this JSON API and fetch every product's detail endpoint with ExtractHub."
- "Run my ExtractHub pipeline every morning at 6 UTC" (after a test run).

## What is inside

```
extracthub-pipeline-builder/
├── SKILL.md                          workflow the agent follows
├── references/
│   ├── chain-design.md               list -> detail chains, url_output_field, url_template, fan-out cap, pins
│   ├── costs-and-runs.md             what a run costs, estimates, maxCostPerRun, job status, results
│   └── schedules-and-webhooks.md     cron schedules, the user-OK rule, webhooks and their secret
└── examples/                         create_pipeline inputs + notes
```

It needs the ExtractHub MCP server (the pipeline tools) and works together with the
[extracthub-extractor-builder](https://github.com/extracthub-io/extracthub-extractor-builder) skill, which
builds any extractor a step still needs. Install both.

## Install

### Claude Code
Clone both skills into your personal skills directory (or a project's `.claude/skills/`):

```bash
git clone https://github.com/extracthub-io/extracthub-pipeline-builder.git ~/.claude/skills/extracthub-pipeline-builder
git clone https://github.com/extracthub-io/extracthub-extractor-builder.git ~/.claude/skills/extracthub-extractor-builder
```

### Other agents
Agents that support the Agent Skills format load the folder from their skills directory. For any other
agent, give it `SKILL.md` as instructions and keep the folder next to it so it can open the references.

## Connect the ExtractHub MCP server
Same server as the extractor builder skill: `https://api.extracthub.io/mcp` (Streamable HTTP) with your API key
as `Authorization: Bearer <key>`. Setup snippets for Claude Code, Cursor and Claude Desktop:
https://extracthub.io/mcp. Keep the key in the client configuration or an environment variable, never in a
committed file.

A test key (`exth_test_...`) can validate and read pipelines; creating, updating, running and cancelling need a
live key (pipelines have no sandbox). Running a pipeline spends credit.

## Tools used
`search_extractors`, `get_extractor`, `get_account`, `validate_pipeline`, `create_pipeline`, `update_pipeline`,
`get_pipeline`, `list_pipelines`, `run_pipeline`, `get_pipeline_job`, `cancel_pipeline_job`.
