# daily_ai_papers

Daily AI news and paper digests.

Structure
- news/YYYY/MM/YYYY-MM-DD.md: daily AI news digest
- papers/YYYY/MM/YYYY-MM-DD.md: daily AI papers digest

Latest
- news/2026/04/2026-04-20.md
- papers/2026/04/2026-04-20.md

Task description

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
- Prefer reputable live sources such as:
  - OpenAI
  - Anthropic
  - Google / Google DeepMind
  - Hugging Face
  - TechCrunch AI
  - The Verge AI
- Prefer important items from roughly the last 1-3 days.
- Focus on meaningful launches, model releases, tooling updates, research-product announcements, infrastructure, safety, agents, robotics, or ecosystem moves.

Papers digest requirements
- Create 6-10 notable recent arXiv papers.
- Prefer papers from roughly the last 1-3 days.
- Focus on topics such as:
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

Required markdown format

News file:
- Title line exactly:
  - # AI News Digest — $DATE
- Subtitle line exactly:
  - Curated from recent live sources.
- Then sections in this order:
  - ## Overview
  - ## News
  - ## Themes
- Under ## News, each item must use this structure:
  - ### N. <headline>
  - Date: <YYYY-MM-DD>
  - Source: <source>
  - URL: <url>
  - Summary: <2-3 sentence explanation of why it matters>
- The Overview should be 2-4 sentences summarizing the day as a whole.
- The Themes section should contain 3-5 bullet points.

Papers file:
- Title line exactly:
  - # AI Papers Digest — $DATE
- Subtitle line exactly:
  - Selected recent arXiv papers.
- Then sections in this order:
  - ## Overview
  - ## Papers
  - ## Themes
- Under ## Papers, each item must use this structure:
  - ### N. <paper title>
  - Date: <YYYY-MM-DD>
  - URL: <arXiv url>
  - Summary: <2-3 sentence explanation of why it matters>
- The Overview should be 2-4 sentences summarizing the paper set as a whole.
- The Themes section should contain 3-5 bullet points.

Quality bar
- Keep markdown neat and consistent.
- Every item must have its own Summary field.
- Summaries should explain significance, not just restate the title.
- Prefer signal over volume.
- Avoid duplicate items or repetitive phrasing.

Repository update rules
- Create parent directories as needed.
- Overwrite the current day's files if they already exist.
- Update the Latest section in this README to point to the new files.
- Keep the Structure section unchanged unless the repository layout changes.

Git and environment rules
- Work in this repository root.
- Before network operations, use the environment's configured proxy settings if required.
- Do not store internal proxy endpoints in repository documentation.
- Use git over SSH with:
  - GIT_SSH_COMMAND='ssh -F /opt/data/home/.ssh/config'
- If git user.name or user.email is not set in the repo, use:
  - Hermes Agent
  - hermes-github@6739d394701d

Commit behavior
- Commit all changed files with message:
  - docs: add $DATE AI news and papers digest
- Push to origin main.
- If there are no content changes, do not fail; report that nothing changed.

Validation checklist
- Confirm both files exist for the target date.
- Confirm README Latest points to the same date.
- Confirm git status is clean after commit.
- Confirm push to origin main succeeded.
