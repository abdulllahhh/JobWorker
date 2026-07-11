# LinkedIn .NET Job Watcher — GitHub Actions Setup Guide

This script checks LinkedIn every ~5 minutes for new .NET / C# developer jobs
in Egypt and sends you a Telegram notification when one is found. It now runs
entirely on **GitHub Actions** — no server, no Railway, no credit card needed.

**How it works now:** instead of one process running forever, GitHub spins up
a fresh runner every 5 minutes on a schedule, it does one job-scan, sends any
Telegram alerts, saves progress back into your repo (`seen_jobs.json`), then
shuts down. Repeat forever, for free.

**What you need:**
1. A Telegram bot (to receive notifications)
2. A GitHub repo containing these files
3. Two GitHub Secrets (Telegram token + chat ID) — no GitHub personal access
   token needed this time, GitHub Actions provides one automatically.

---

## PART 1 — Create Your Telegram Bot

*(Skip this if you already have a bot token and chat ID from before — they still work.)*

1. Open Telegram, search for **@BotFather**, open it.
2. Send `/newbot`, give it a display name, then a username ending in `bot`.
3. BotFather replies with a **bot token** — looks like `7412563890:AAFh...`. Copy it.
4. Message your new bot (search its username, press **Start**, send `hello`).
5. In your browser, open (replacing `YOUR_TOKEN`):
   ```
   https://api.telegram.org/botYOUR_TOKEN/getUpdates
   ```
6. Find `"chat": { "id": 123456789, ... }` in the response. That number is your **Chat ID**.

> If the response is empty, send another message to the bot and refresh the URL.

---

## PART 2 — Upload the Files to GitHub

1. Go to [github.com](https://github.com) and create a **private** repository (e.g. `job-watcher`), no README.
2. Upload these files/folders exactly as they are in this project:
   ```
   job-watcher/
   ├── watcher.py
   ├── requirements.txt
   ├── seen_jobs.json
   └── .github/
       └── workflows/
           └── job-watcher.yml
   ```
   Tip: on the repo page, use **Add file → Upload files**, and drag the whole
   folder in — GitHub will preserve the `.github/workflows/` path automatically.
3. Click **Commit changes**.

> You do **not** need `Procfile` or the old `deploy.yml` — those were Railway-specific and have been moved into `old_railway_setup/` for reference only.

---

## PART 3 — Add Your Telegram Secrets

GitHub Actions needs your Telegram credentials, stored securely as **repo secrets**
(never visible in logs or to anyone browsing your repo).

1. In your repo, go to **Settings** → **Secrets and variables** → **Actions**.
2. Click **New repository secret**.
3. Add:
   | Name | Value |
   |---|---|
   | `TELEGRAM_TOKEN` | your bot token from Part 1 |
   | `TELEGRAM_CHAT_ID` | your chat ID from Part 1 |
4. Click **Add secret** after each one.

That's it — no `GITHUB_TOKEN` secret to create. GitHub Actions automatically
injects a short-lived token with repo access into every workflow run; the
workflow file is already configured to use it (`permissions: contents: write`).

---

## PART 4 — Enable Scheduled Runs

GitHub sometimes disables scheduled workflows on upload until you interact
with them once. To be safe:

1. Go to your repo's **Actions** tab.
2. If prompted, click **"I understand my workflows, go ahead and enable them"**.
3. Click on **LinkedIn Job Watcher** in the left sidebar.
4. Click **Run workflow** (top right) → **Run workflow** to trigger a manual first run.
5. Click into the run to watch the logs live. You should see:
   ```
   LinkedIn Job Watcher — Egypt | .NET & C# jobs only (single run mode)
   [12:00:00] First run — seeding seen jobs (no notifications)...
     47 recent listing(s) found.
     (seed) .NET Developer @ Some Company
     ...
   ```
6. From now on, it will automatically run every ~5 minutes on the `cron` schedule —
   no manual trigger needed. You can always re-trigger manually from this same page.

---

## PART 5 — Verify Telegram is Working

Send `/status` to your bot in Telegram. Within ~5 minutes (the next scheduled run),
you should get a reply like:

```
✅ Watcher is alive!

⚙️ Running on GitHub Actions (checks every ~5 min)
🕐 This check: 12:05:00
📍 Watching: Egypt | .NET & C# jobs only
```

(This reply only appears on the *next* run after you send the command, since
there's no longer a process listening 24/7 — it checks once per cycle instead.)

---

## Important Notes About This Setup

- **Timing isn't exact.** GitHub's cron scheduler can occasionally delay a run
  by a few minutes under heavy platform load. Fine for job alerts, not fine
  for anything requiring precise timing.
- **Inactive repos get scheduled workflows paused.** If a repo has zero commits
  for 60 days, GitHub automatically disables its scheduled workflows. Since this
  script commits `seen_jobs.json` whenever it finds new listings, this is
  unlikely to happen in practice — but if your bot ever goes quiet for a long
  stretch, check the **Actions** tab and re-enable if needed.
- **Free minutes:** private repos get 2,000 free Actions minutes/month. Each
  run of this script takes well under a minute, so 5-minute polling uses a
  small fraction of that allowance. If you'd rather have zero limit, you can
  make the repo public instead (Actions minutes are unlimited on public repos).

---

## How to Block a Spammy Company

Open `watcher.py`, find:

```python
BLOCKED_COMPANIES = {
    "bairesdev",
    "micro1",
    "jobs ai",
}
```

Add the company name, lowercase, exactly as it appears on LinkedIn:

```python
BLOCKED_COMPANIES = {
    "bairesdev",
    "micro1",
    "jobs ai",
    "new spammy company",
}
```

Commit the change — the next scheduled run will use the updated list automatically.
No redeploy step needed, unlike the old Railway setup.

---

## Files in Your Repo (Summary)

```
job-watcher/
├── watcher.py                        ← runs one check cycle, then exits
├── requirements.txt                  ← Python packages
├── seen_jobs.json                    ← starts as [] — filled in automatically
└── .github/
    └── workflows/
        └── job-watcher.yml           ← the cron schedule + run steps
```
