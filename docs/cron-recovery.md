# Cron Task Recovery

This file records the current cron setup for this repo and how to rebuild it if Hermes runtime state is lost.

## Source of truth

This document is based on:
- `README.md`
- current cron state from `cronjob list`
- `/opt/data/cron/jobs.json`
- local helper script: `~/.hermes/scripts/check_learning_plan_submission.py`

## Active scheduled tasks

### 1. `daily-ai-news-papers-digest`

**Purpose**
- Generate the daily AI news digest and daily AI papers digest.
- Update `README.md` Latest links.
- Commit and push to `origin/main`.

**Current configuration**
- Job id: `cdc68a300119`
- Schedule: `0 0 * * *`
- Intended meaning: `08:00 Asia/Shanghai`
- Model/provider: `copilot` + `gpt-5.4`
- Delivery target: `slack:C0AUJ44J97W`
- Skills:
  - `daily-ai-digest-repo`
  - `arxiv`
  - `github-repo-management`

### 2. `weekly-learning-review`

**Purpose**
- Every Saturday morning, summarize this week's daily digests.
- Send the weekly recap in Gordon's academic-coach style for Jerome.
- Remind Jerome to submit next week's learning plan before 20:00 that night.

**Current configuration**
- Job id: `bb65e1abb2dd`
- Schedule: `0 9 * * 6`
- Intended meaning: every Saturday `09:00 Asia/Shanghai`
- Model/provider: `copilot` + `gpt-5.4`
- Delivery target: `slack:C0AUJ44J97W`
- Skills: none

### 3. Saturday evening learning-plan reminder chain

This workflow is implemented as **7 separate cron jobs**, one every 30 minutes from 20:00 to 23:00 on Saturdays.

**Shared behavior**
- Delivery target: `slack:C0AUJ44J97W`
- Model/provider: `copilot` + `gpt-5.4`
- Shared pre-run script: `check_learning_plan_submission.py`
- Script path at runtime: `~/.hermes/scripts/check_learning_plan_submission.py`

**Current jobs**
- `learning-plan-reminder-2000` — job id `d3f153b23a5c` — schedule `0 20 * * 6`
- `learning-plan-reminder-2030` — job id `1ba4fd8ddf1d` — schedule `30 20 * * 6`
- `learning-plan-reminder-2100` — job id `fd4397cada75` — schedule `0 21 * * 6`
- `learning-plan-reminder-2130` — job id `213642fa28e4` — schedule `30 21 * * 6`
- `learning-plan-reminder-2200` — job id `907cb1649bcb` — schedule `0 22 * * 6`
- `learning-plan-reminder-2230` — job id `0f01111815a5` — schedule `30 22 * * 6`
- `learning-plan-reminder-2300` — job id `041783fa760b` — schedule `0 23 * * 6`

> Job IDs are runtime artifacts. Recreated jobs will get new IDs.

## Exact task intent

### Daily digest task
- Work in repo: `/opt/data/home/daily_ai_papers`
- Before writing or summarizing for Slack, read:
  - `docs/assistant-prompts/jerome-academic-coach.md`
- Use Asia/Shanghai date basis:
  - `DATE=$(TZ=Asia/Shanghai date +%F)`
  - `YEAR=$(TZ=Asia/Shanghai date +%Y)`
  - `MONTH=$(TZ=Asia/Shanghai date +%m)`
- Write:
  - `news/YYYY/MM/YYYY-MM-DD.md`
  - `papers/YYYY/MM/YYYY-MM-DD.md`
- Keep the markdown format defined in `README.md`
- Update `README.md` Latest links
- Commit with:
  - `docs: add $DATE AI news and papers digest`
- Push to `origin main`
- After the repo update, send a concise Gordon-style Slack note for Jerome with:
  - a brief confirmation that the digest is ready
  - 2-3 `今日 AI 观察` bullets
  - one short `Gordon 提醒` linking an AI trend to a concrete study habit

### Weekly Saturday morning review
- Work in repo: `/opt/data/home/daily_ai_papers`
- Read this week's available daily digest files under:
  - `news/YYYY/MM/YYYY-MM-DD.md`
  - `papers/YYYY/MM/YYYY-MM-DD.md`
- Read `docs/assistant-prompts/jerome-academic-coach.md` before drafting the Slack message
- Produce a concise weekly review in Gordon's academic-coach tone for Jerome
- Style should stay short, encouraging, and logically clear
- Include:
  - 3-5 bullet points on the week's AI learning/industry highlights
  - a `Gordon 点评` section explaining what Jerome can learn from these trends
  - a gentle reminder that next week's learning plan should be sent before `20:00` that night
  - 2-3 starter questions to help Jerome write the plan

### Saturday evening reminder rule
Current default detection rule:
- If Johnson sends a Slack user message on Saturday between `20:00` and `23:59` containing either:
  - `学习规划`
  - or a phrase matching `下周...规划`
- then the system treats the learning plan as received

If `PLAN_RECEIVED=false`, the reminder job sends a short Gordon-style prompt that lowers the writing barrier by offering concrete starter questions.
If `PLAN_RECEIVED=true`, the reminder job sends only a short confirmation that the plan has already been received.

## Recovery prerequisites

Before recreating jobs:

1. Verify repo remote:
```bash
git -C /opt/data/home/daily_ai_papers remote -v
```

2. Verify cron/config readability:
```bash
stat /opt/data/cron/jobs.json /opt/data/config.yaml
```

3. Verify runtime user:
```bash
whoami && id
```

4. Verify helper script directory exists:
```bash
mkdir -p /opt/data/scripts
```

## Recovery helper script

Create this file first:
- `/opt/data/scripts/check_learning_plan_submission.py`

Script content:

```python
#!/usr/bin/env python3
import sqlite3
import re
from datetime import datetime, time
from zoneinfo import ZoneInfo

DB = '/opt/data/state.db'
USER_ID = 'U04BS0392HH'
TZ = ZoneInfo('Asia/Shanghai')
KEYWORDS = [
    re.compile(r'学习规划'),
    re.compile(r'下周.{0,20}规划'),
]

now = datetime.now(TZ)
start = datetime.combine(now.date(), time(20, 0), TZ)
end = datetime.combine(now.date(), time(23, 59, 59), TZ)

conn = sqlite3.connect(DB)
cur = conn.cursor()
cur.execute(
    """
    SELECT m.content, m.timestamp, m.session_id
    FROM messages m
    JOIN sessions s ON s.id = m.session_id
    WHERE s.source = 'slack'
      AND s.user_id = ?
      AND m.role = 'user'
      AND m.timestamp >= ?
      AND m.timestamp <= ?
    ORDER BY m.timestamp DESC
    """,
    (USER_ID, start.timestamp(), end.timestamp()),
)
rows = cur.fetchall()

matches = []
for content, ts, session_id in rows:
    text = content or ''
    if any(p.search(text) for p in KEYWORDS):
        matches.append((text, ts, session_id))

print(f"NOW={now.isoformat()}")
print(f"WINDOW_START={start.isoformat()}")
print(f"WINDOW_END={end.isoformat()}")
print(f"TOTAL_USER_MESSAGES={len(rows)}")
print(f"MATCH_COUNT={len(matches)}")
print(f"PLAN_RECEIVED={'true' if matches else 'false'}")
if matches:
    latest_text, latest_ts, latest_session = matches[0]
    latest_dt = datetime.fromtimestamp(latest_ts, TZ)
    compact = re.sub(r'\s+', ' ', latest_text).strip()
    if len(compact) > 200:
        compact = compact[:200] + '...'
    print(f"LATEST_MATCH_TIME={latest_dt.isoformat()}")
    print(f"LATEST_MATCH_SESSION={latest_session}")
    print(f"LATEST_MATCH_TEXT={compact}")
```

Recommended post-create step:
```bash
chmod +x /opt/data/scripts/check_learning_plan_submission.py
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
  "prompt": "Work in the GitHub repo at /opt/data/home/daily_ai_papers. Before writing, read /opt/data/home/daily_ai_papers/README.md, /opt/data/home/daily_ai_papers/docs/cron-recovery.md, and /opt/data/home/daily_ai_papers/docs/assistant-prompts/jerome-academic-coach.md. Use Asia/Shanghai as the date basis. Write the daily digest files to news/$YEAR/$MONTH/$DATE.md and papers/$YEAR/$MONTH/$DATE.md, update README.md Latest, follow the exact formatting requirements in README.md and docs/cron-recovery.md, commit with message docs: add $DATE AI news and papers digest, and push to origin main. Use the environment's configured proxy settings if required, use git over SSH with ssh config at /opt/data/home/.ssh/config, and fall back to Python stdlib or Hermes web tools if curl or a source feed is unavailable. After the repo update, send a concise final Slack message in Gordon's style for Jerome: first confirm the digest is ready, then list 2-3 '今日 AI 观察' bullets based on the day's highest-signal items, then add one short 'Gordon 提醒' that connects one AI trend to a concrete study habit for a middle-school student. Tone: patient, encouraging, logical, not childish, not verbose."
}
```

### Recreate the weekly Saturday morning review

```json
{
  "action": "create",
  "name": "weekly-learning-review",
  "schedule": "0 9 * * 6",
  "deliver": "slack:C0AUJ44J97W",
  "model": {"provider": "copilot", "model": "gpt-5.4"},
  "prompt": "今天是周六晨间周报时间。先读取 /opt/data/home/daily_ai_papers/docs/assistant-prompts/jerome-academic-coach.md 与 /opt/data/home/daily_ai_papers/docs/cron-recovery.md。再在 GitHub 仓库 /opt/data/home/daily_ai_papers 中回顾本周日报内容，优先读取本周已有的 news/YYYY/MM/YYYY-MM-DD.md 与 papers/YYYY/MM/YYYY-MM-DD.md。请以 Gordon 面向 Jerome 的风格输出一份简洁周报，要求：1) 标题可直接写“本周 AI 周报”；2) 用 3-5 条 bullet 总结本周最值得 Jerome 关注的 AI 学习/行业重点；3) 增加一个“Gordon 点评”小段，用中学生能懂的话解释这些变化对学习方法、思维方式或未来能力有什么启发；4) 增加一个“周计划提醒”小段，温和提醒 Jerome 在今晚 20:00 前发送下周学习规划，并给出 2-3 个引导问题帮助他开头；5) 风格耐心、鼓励、逻辑清晰、不要太啰嗦；6) 直接发送到 Slack。"
}
```

### Recreate the evening reminder chain

Restore these 7 jobs using the same model/provider and delivery target, all with:
- `script: "check_learning_plan_submission.py"`
- `deliver: "slack:C0AUJ44J97W"`
- `model: {"provider":"copilot","model":"gpt-5.4"}`

Example for 20:00:

```json
{
  "action": "create",
  "name": "learning-plan-reminder-2000",
  "schedule": "0 20 * * 6",
  "deliver": "slack:C0AUJ44J97W",
  "model": {"provider": "copilot", "model": "gpt-5.4"},
  "script": "check_learning_plan_submission.py",
  "prompt": "你正在执行周六晚间学习规划提醒。先读取预运行脚本注入的上下文，再遵循 /opt/data/home/daily_ai_papers/docs/assistant-prompts/jerome-academic-coach.md 的 Gordon 风格。如果上下文里显示 PLAN_RECEIVED=true，则只输出：Gordon 收到啦，本周学习规划已记下，今晚不再提醒。 如果 PLAN_RECEIVED=false，则只输出：Gordon 来提醒一下：如果下周学习规划还没写，可以先回这 3 个小问题：1. 下周最想补哪一科？2. 最容易卡住的题型是什么？3. 每天准备学多久？写几行发来就可以。"
}
```

The remaining schedules are:
- `30 20 * * 6`
- `0 21 * * 6`
- `30 21 * * 6`
- `0 22 * * 6`
- `30 22 * * 6`
- `0 23 * * 6`

Use corresponding names:
- `learning-plan-reminder-2030`
- `learning-plan-reminder-2100`
- `learning-plan-reminder-2130`
- `learning-plan-reminder-2200`
- `learning-plan-reminder-2230`
- `learning-plan-reminder-2300`

## Slack delivery notes

Current delivery target for all active jobs:
- `slack:C0AUJ44J97W`

If delivery fails with `channel_not_found` or `not_in_channel`:
- verify the bot is actually in the channel
- if needed, switch to a known-good DM/thread target

## Validation after recovery

### Daily digest
1. `cronjob(action='list')` shows the job
2. manual run succeeds
3. new session/output artifacts appear
4. repo files update correctly
5. Slack delivery is visible

### Weekly review
1. `cronjob(action='list')` shows `weekly-learning-review`
2. manual run produces a concise weekly summary
3. Slack delivery is visible

### Evening reminder chain
1. `cronjob(action='list')` shows all 7 reminder jobs
2. helper script exists and runs
3. manual run of one reminder job shows `PLAN_RECEIVED=true/false` in injected script context
4. reminder behavior matches actual Saturday evening Slack messages

## Operational pitfalls

### 1. Cron/config permissions
If `/opt/data/cron/jobs.json` or `/opt/data/config.yaml` is unreadable by the Hermes runtime user, listing/running jobs may fail.

### 2. Job-level provider/model drift
Do not assume recreated jobs inherit the current default model. Set provider/model explicitly.

### 3. Slack delivery can fail after task execution
A task may run successfully but still fail at the final send step. Check session/output artifacts separately from delivery logs.

### 4. Evening reminders depend on local state history
The reminder-chain condition depends on `/opt/data/state.db` containing current Slack messages and being readable.

### 5. Current learning-plan detection is keyword-based
If the wording convention changes, update `check_learning_plan_submission.py` too.

### 6. Keep public docs free of internal infrastructure details
Use wording like "use the environment's configured proxy settings if required" instead of storing internal proxy endpoints.

## Recovery checklist
1. Restore the repo
2. Verify git SSH access
3. Verify Hermes cron/config permissions
4. Restore `check_learning_plan_submission.py`
5. Recreate the daily digest job
6. Recreate the weekly Saturday morning review
7. Recreate the 7 evening reminder jobs
8. Manually test at least one morning job and one evening job
9. Confirm Slack delivery
10. Update this file whenever job names, schedules, prompts, delivery target, or detection rules change
