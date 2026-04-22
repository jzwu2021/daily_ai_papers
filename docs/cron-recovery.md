# Cron Task Recovery

This file records the current Hermes scheduled tasks for this repo and how to recreate them if Hermes runtime state is lost.

## Source of truth

This document is based on:
- `README.md`
- current cron state in `/opt/data/cron/jobs.json`
- successful and failed cron run artifacts under `/opt/data/sessions/` and `/opt/data/cron/output/`

## Active scheduled tasks

### 1. `daily-ai-news-papers-digest`

**Purpose**
- Generate one daily AI news digest and one daily AI papers digest.
- Use the Asia/Shanghai calendar day as the date basis.
- Write the files into this repo, update `README.md`, commit, and push.

**Current known configuration**
- Job id: `cdc68a300119`
- Schedule: `0 0 * * *`
- Intended wall-clock meaning: `08:00 Asia/Shanghai`
- Model/provider: `copilot` + `gpt-5.4`
- Delivery target: `slack:C0AUJ44J97W`
- Skills:
  - `daily-ai-digest-repo`
  - `arxiv`
  - `github-repo-management`

> Job IDs are runtime artifacts. Hermes will assign a new id when recreated.

### 2. `A股优秀管理层专题季度复盘提醒`

**Purpose**
- Remind Johnson to review the `topics/a-share-management/` topic in this repo.
- Check the latest file under `topics/a-share-management/治理评价/`.
- Create a new same-day governance review file.
- Update `topics/a-share-management/TODO.md` with the next quarterly review item.
- Report the reminder result back to Slack.

**Current known configuration**
- Job id: `b6e5475d01d7`
- Current schedule: `once at 2026-07-20 09:00`
- Intended timezone: `Asia/Shanghai`
- Model/provider: `copilot` + `gpt-5.4`
- Delivery target: `slack:C0AUJ44J97W`
- Skills: none

> This task is currently configured as a one-shot reminder, not a recurring quarterly cron expression.

## Exact task intent

### Daily digest outputs
For the current Shanghai date, write:
- `news/YYYY/MM/YYYY-MM-DD.md`
- `papers/YYYY/MM/YYYY-MM-DD.md`

Use:
- `DATE=$(TZ=Asia/Shanghai date +%F)`
- `YEAR=$(TZ=Asia/Shanghai date +%Y)`
- `MONTH=$(TZ=Asia/Shanghai date +%m)`

### Daily digest content rules
- News: 5-8 high-signal items from reputable recent sources.
- Papers: 6-10 recent notable arXiv papers.
- Keep the exact markdown structure documented in `README.md`.
- Update `README.md` Latest links.
- Commit with:
  - `docs: add $DATE AI news and papers digest`
- Push to `origin main`.

### A-share quarterly review reminder rules
- Work in repo: `/opt/data/home/daily_ai_papers`
- Focus path:
  - `topics/a-share-management/`
  - `topics/a-share-management/治理评价/`
  - `topics/a-share-management/TODO.md`
- Review the latest governance-evaluation file.
- Create a new review file with the latest date.
- Refresh TODO so the next quarterly review is scheduled.
- Send a concise reminder/result message to Slack.

## Recovery prerequisites

Before recreating either task:

1. Verify repo exists and remote is correct:
   ```bash
   git -C /opt/data/home/daily_ai_papers remote -v
   ```
2. Verify Hermes can read cron state:
   ```bash
   stat /opt/data/cron/jobs.json /opt/data/config.yaml
   ```
3. Verify Hermes runtime user:
   ```bash
   whoami && id
   ```
4. If needed, verify recent session artifacts:
   ```bash
   ls -1t /opt/data/sessions/session_cron_* | head
   ```

## Hermes recreation commands

### Recreate the daily digest job

```json
{
  "action": "create",
  "name": "daily-ai-news-papers-digest",
  "schedule": "0 0 * * *",
  "deliver": "slack:C0AUJ44J97W",
  "model": {"provider": "copilot", "model": "gpt-5.4"},
  "skills": ["daily-ai-digest-repo", "arxiv", "github-repo-management"],
  "prompt": "Work in the GitHub repo at /opt/data/home/daily_ai_papers. Use Asia/Shanghai as the date basis. Write the daily digest files to news/$YEAR/$MONTH/$DATE.md and papers/$YEAR/$MONTH/$DATE.md, update README.md Latest, follow the exact formatting requirements in README.md and docs/cron-recovery.md, commit with message docs: add $DATE AI news and papers digest, and push to origin main. Use the environment's configured proxy settings if required, use git over SSH with ssh config at /opt/data/home/.ssh/config, and fall back to Python stdlib or Hermes web tools if curl or a source feed is unavailable." 
}
```

### Recreate the A-share quarterly review reminder

Use the next intended quarterly review timestamp in Asia/Shanghai.

```json
{
  "action": "create",
  "name": "A股优秀管理层专题季度复盘提醒",
  "schedule": "2026-07-20T09:00:00+08:00",
  "repeat": 1,
  "deliver": "slack:C0AUJ44J97W",
  "model": {"provider": "copilot", "model": "gpt-5.4"},
  "prompt": "提醒 Johnson：今天需要对 GitHub 仓库 daily_ai_papers 中 topics/a-share-management/ 这个专题做季度复盘。请检查 topics/a-share-management/治理评价/ 下最近一次评估文件，按最新日期新建一份当日治理评价文件，复核 A 股优秀管理层 Top50，重点关注治理、信披、IR、监管处罚、高管变动、资本配置与经营质量变化，并更新 topics/a-share-management/TODO.md 中下一次 3 个月后的复盘事项。最后向 Slack 汇报本次复盘提醒结果。"
}
```

## Slack delivery recovery notes

Current configured delivery target for both tasks:
- `slack:C0AUJ44J97W`

Important notes:
- Prefer explicit Slack targets over `deliver=origin`.
- If Slack delivery fails with `channel_not_found` or `not_in_channel`, verify that the bot is actually in the target channel.
- If channel delivery remains unreliable, fall back to a known-good DM or thread target.

## Validation after recreation

### Daily digest job
1. `cronjob(action='list')` shows the recreated job.
2. `cronjob(action='run', job_id=...)` succeeds.
3. A new file appears under:
   - `/opt/data/sessions/session_cron_<jobid>_*.json`
4. A new output artifact appears under:
   - `/opt/data/cron/output/<jobid>/`
5. Repo files for the target date exist and `README.md` Latest is updated.
6. Git push succeeds.
7. Confirm Slack delivery in the target channel.

### Quarterly reminder job
1. `cronjob(action='list')` shows the recreated job.
2. If using a one-shot schedule, confirm `next_run_at` matches the intended quarter date.
3. For a manual test, run it once with `cronjob(action='run', job_id=...)`.
4. Confirm the governance-evaluation file and `TODO.md` were updated as intended.
5. Confirm Slack delivery in the target channel.

## Operational pitfalls

### 1. Cron permissions
If `/opt/data/cron/jobs.json` is unreadable by the Hermes runtime user, jobs may not list or run correctly.

### 2. Config permissions
If `/opt/data/config.yaml` is unreadable, scheduler/runtime behavior can break.

### 3. Job-level provider/model drift
Do not assume old jobs inherit the current default model. If a recovered job shows null provider/model, set them explicitly.

### 4. Slack delivery errors can be delivery-only
A task may execute successfully but still fail at the final Slack send step. Check session files and output artifacts separately from delivery logs.

### 5. Do not publish internal infrastructure details
Keep repo docs generic. Say "use the environment's configured proxy settings if required" rather than storing internal proxy endpoints.

## Recovery checklist
1. Restore the repo.
2. Verify git SSH access.
3. Verify Hermes cron/config permissions.
4. Recreate both jobs with explicit model/provider and explicit Slack delivery target.
5. Manually run both jobs once.
6. Confirm session/output artifacts.
7. Confirm message delivery in Slack.
8. Update this document whenever job name, schedule, prompt, skills, or delivery target changes.
