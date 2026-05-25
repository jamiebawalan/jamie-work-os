# Jamie's Work OS

Personal productivity system for Jamie Bawalan, Principal at BCG Manila.

## What this is

A self-contained web app that runs entirely in the browser. No backend, no login, no server. All data stored in `localStorage`.

## Files

- `index.html` — the full Work OS app
- `tasks.json` — N8N bridge file. When the Telegram capture bot extracts tasks, N8N writes them here. The app reads this file on load and merges new tasks into localStorage.

## Modules

- **Today** — stats, week ahead, urgent tasks, neglect radar
- **Tasks** — three urgency lanes (Urgent / Important / Background), drag-to-reprioritise, filter by project, clear done
- **Milestones** — sorted by date, colour-coded by workstream
- **Projects** — open task counts and next steps per account
- **People** — Constellation (partners/MDPs) and Pyramid (below-cohort)
- **Capture** — paste Granola/meeting notes, copy prompts to Claude
- **Inbox** — copy Outlook triage prompts to Claude
- **Check-in** — fragmentation index, cognitive margin, 5 lifestyle sliders, 14-day sparkline history

## N8N bridge

When N8N captures a task via Telegram, it POSTs to the GitHub API to update `tasks.json`. The app fetches this file on every load and merges any new tasks (by ID) into localStorage. Tasks already in localStorage are never overwritten — done states and urgency changes persist.

## Deployment

Hosted on GitHub Pages. Go to Settings → Pages → Deploy from branch → main → / (root).

URL: `https://jamiebawalan.github.io/jamie-work-os`
