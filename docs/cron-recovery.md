# Cron Task Recovery

This file documents the scheduled tasks and the operational details needed to recreate them if the Hermes runtime state is lost.

## Source of truth

This recovery document is based on:
- the repository task description in `README.md`
- recorded Hermes cron runs and prior session history
- the currently known production job configuration recovered from session logs

## Active scheduled task

### 1. daily-ai-news-papers-digest

**Purpose**
- Generate one daily AI news digest and one daily AI papers digest.
- Use the Asia/Shanghai calendar day as the date basis.
- Commit the generated markdown files into this repository and push to `origin/main`.

**Current known schedule**
- Cron expression: `0 0 * * *`
- Intended wall-clock meaning: `08:00 Asia/Shanghai`
- Reason: the Hermes scheduler in this environment was observed to interpret cron expressions in UTC, so `00:00 UTC` corresponds to `08:00 Asia/Shanghai`.

**Known current job metadata**
- Name: `daily-ai-news-papers-digest`
- Replacement job id used previously: `cdc68a300119`
- Historical broken job id (do not reuse): `fa9b4abafa2b`

> Job IDs are runtime artifacts. When recreating the task, Hermes will assign a new job id automatically.

## Exact task intent

### Output files
For the current Shanghai date:
- `news/YYYY/MM/YYYY-MM-DD.md`
- `papers/YYYY/MM/YYYY-MM-DD.md`

### Date calculation
Use:
- `DATE=$(TZ=Asia/Shanghai date +%F)`
- `YEAR=$(TZ=Asia/Shanghai date +%Y)`
- `MONTH=$(TZ=Asia/Shanghai date +%m)`

### News digest requirements
- Create 5-8 high-signal AI news items.
- Prefer reputable live sources such as:
  - OpenAI
  - Anthropic
  - Google / Google DeepMind
  - Hugging Face
  - TechCrunch AI
  - The Verge AI
- Prefer important items from roughly the last 1-3 days.
- Focus on launches, model releases, tooling updates, research-product announcements, infrastructure, safety, agents, robotics, and ecosystem moves.

### Papers digest requirements
- Create 6-10 notable recent arXiv papers.
- Prefer papers from roughly the last 1-3 days.
- Focus on:
  - LLMs
  - reasoning
  - agents
  - multimodal systems
  - vision
  - audio
  - robotics
  - inference
  - benchmarks
  - safety / alignment
- Avoid withdrawn papers if detectable.

## Required markdown format

### News file
- Title: `# AI News Digest — $DATE`
- Subtitle: `Curated from recent live sources.`
- Section order:
  - `## Overview`
  - `## News`
  - `## Themes`
- Each item under `## News` must be:
  - `### N. <headline>`
  - `Date: <YYYY-MM-DD>`
  - `Source: <source>`
  - `URL: <url>`
  - `Summary: <2-3 sentence explanation of why it matters>`

### Papers file
- Title: `# AI Papers Digest — $DATE`
- Subtitle: `Selected recent arXiv papers.`
- Section order:
  - `## Overview`
  - `## Papers`
  - `## Themes`
- Each item under `## Papers` must be:
  - `### N. <paper title>`
  - `Date: <YYYY-MM-DD>`
  - `URL: <arXiv url>`
  - `Summary: <2-3 sentence explanation of why it matters>`

## Repository update rules
- Create parent directories as needed.
- Overwrite the current day's files if they already exist.
- Update the `Latest` section in `README.md` to point to the new files.
- Keep the repository structure unchanged unless the layout itself changes.

## Git and environment requirements
Before network operations, export:
- `HTTP_PROXY=http://proxy.ims.intel.com:911`
- `HTTPS_PROXY=http://proxy.ims.intel.com:911`
- `NO_PROXY=localhost,127.0.0.1,.local,192.168.*,*.intel.com`

Use git over SSH with:
- `GIT_SSH_COMMAND='ssh -F /opt/data/home/.ssh/config'`

If repo-local git identity is missing, use:
- `user.name=Hermes Agent`
- `user.email=hermes-github@6739d394701d`

## Commit behavior
- Commit all changed files with message:
  - `docs: add $DATE AI news and papers digest`
- Push to `origin main`.
- If there are no content changes, report success without failing.

## Hermes recreation command

Use the following command shape to recreate the scheduled task from Hermes:

```json
{
  "action": "create",
  "name": "daily-ai-news-papers-digest",
  "schedule": "0 0 * * *",
  "deliver": "local",
  "model": {"provider": "copilot", "model": "gpt-5.4"},
  "prompt": "Work in repository /opt/data/home/daily_ai_papers. For the current Asia/Shanghai date, generate one AI news digest and one AI papers digest, write them to news/$YEAR/$MONTH/$DATE.md and papers/$YEAR/$MONTH/$DATE.md, update README.md Latest, commit with message docs: add $DATE AI news and papers digest, and push to origin main. Use the formatting and quality requirements documented in README.md and docs/cron-recovery.md.",
  "repeat": 0
}
```

## Recommended recreation prompt

If you need a fuller self-contained prompt for Hermes cron creation, use this:

```text
Work in repository /opt/data/home/daily_ai_papers.

Goal
- Produce one AI news digest and one AI papers digest for the current day.
- Use Asia/Shanghai as the date basis.
- Keep formatting stable so the task can be reproduced consistently by an agent.

Date and paths
- Compute:
  - DATE=$(TZ=Asia/Shanghai date +%F)
  - YEAR=$(TZ=Asia/Shanghai date +%Y)
  - MONTH=$(TZ=Asia/Shanghai date +%m)
- Write files to:
  - news/$YEAR/$MONTH/$DATE.md
  - papers/$YEAR/$MONTH/$DATE.md

News digest requirements
- Create 5-8 high-signal AI news items.
- Prefer reputable live sources such as OpenAI, Anthropic, Google / Google DeepMind, Hugging Face, TechCrunch AI, and The Verge AI.
- Prefer important items from roughly the last 1-3 days.
- Focus on meaningful launches, model releases, tooling updates, research-product announcements, infrastructure, safety, agents, robotics, or ecosystem moves.

Papers digest requirements
- Create 6-10 notable recent arXiv papers.
- Prefer papers from roughly the last 1-3 days.
- Focus on LLMs, reasoning, agents, multimodal systems, vision, audio, robotics, inference, benchmarks, and safety / alignment.
- Avoid withdrawn papers if detectable.

Required markdown format
- News title line exactly: # AI News Digest — $DATE
- News subtitle line exactly: Curated from recent live sources.
- News sections in this order: ## Overview, ## News, ## Themes
- Each news item must include title, date, source, URL, and a 2-3 sentence Summary.
- Papers title line exactly: # AI Papers Digest — $DATE
- Papers subtitle line exactly: Selected recent arXiv papers.
- Papers sections in this order: ## Overview, ## Papers, ## Themes
- Each paper item must include title, date, URL, and a 2-3 sentence Summary.
- Overview sections should be 2-4 sentences.
- Themes sections should contain 3-5 bullet points.

Repository update rules
- Create parent directories as needed.
- Overwrite the current day's files if they already exist.
- Update the Latest section in README.md.

Git and environment rules
- Before network operations, export:
  - HTTP_PROXY=http://proxy.ims.intel.com:911
  - HTTPS_PROXY=http://proxy.ims.intel.com:911
  - NO_PROXY=localhost,127.0.0.1,.local,192.168.*,*.intel.com
- Use git over SSH with:
  - GIT_SSH_COMMAND='ssh -F /opt/data/home/.ssh/config'
- If git user.name or user.email is not set in the repo, use Hermes Agent / hermes-github@6739d394701d.

Commit behavior
- Commit all changed files with message: docs: add $DATE AI news and papers digest
- Push to origin main.
- If there are no content changes, do not fail; report that nothing changed.

Validation checklist
- Confirm both files exist for the target date.
- Confirm README Latest points to the same date.
- Confirm git status is clean after commit.
- Confirm push to origin main succeeded.
```

## Operational pitfalls discovered

### 1. Cron database permissions can prevent all jobs from running
In this Hermes deployment, scheduled jobs failed when:
- `/opt/data/cron/jobs.json` was owned by `root:root`
- file mode was `600`
- Hermes runtime user was `hermes`

Symptom:
- `cronjob list` / `cronjob create` fails with `Permission denied`
- scheduled jobs never fire

Minimal fix:
```bash
sudo chown hermes:hermes /opt/data/cron/jobs.json
sudo chmod 600 /opt/data/cron/jobs.json
```

### 2. Hermes config permissions matter too
If `/opt/data/config.yaml` is unreadable to the `hermes` user, scheduler-related behavior may also fail.

Minimal fix:
```bash
sudo chown hermes:hermes /opt/data/config.yaml
sudo chmod 600 /opt/data/config.yaml
```

### 3. Time drift can make cron appear late or broken
This environment previously showed significant clock skew. Even with a correct cron expression, execution can still happen at the wrong real-world time if the host clock is wrong.

## Recovery checklist
1. Restore this repository.
2. Verify git SSH access works.
3. Verify Hermes runs as the expected user.
4. Verify `/opt/data/config.yaml` and `/opt/data/cron/jobs.json` are readable by that user.
5. Recreate the `daily-ai-news-papers-digest` job with schedule `0 0 * * *`.
6. Run a one-off manual test.
7. Confirm the next scheduled run time is correct relative to UTC and Asia/Shanghai.

## Notes
- This repository currently contains one known production cron task description.
- If more Hermes scheduled tasks are added later, append them to this file using the same format so the repository remains a durable recovery source.
