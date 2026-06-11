# N8N — Granola Meeting Capture Workflow
**Created:** 2026-05-28
**Purpose:** Auto-process Granola meeting notes → tagged tasks (pending.json) + meeting summary (meetings.json)

---

## Architecture
```
Granola note → "Work OS" folder
  → Zapier: Note Added to Folder → POST to N8N webhook
  → N8N webhook receives: title, transcript, action_items, attendees, summary, link
  → N8N Claude node: extract tagged tasks + meeting summary
  → N8N: write tasks to pending.json + summary to meetings.json (GitHub)
  → App Pending tab → approve tasks
  → Claude Desktop folds meetings.json into account MD files (vault sync)
```

---

## ZAPIER SIDE (just a trigger + pipe — no logic)

**Step 1 — Trigger: Granola**
- App: Granola
- Event: **Note Added to Folder**
- Folder: **Work OS** (create this folder in Granola first)

**Step 2 — Action: Webhooks by Zapier**
- Event: **POST**
- URL: [paste N8N webhook URL from the Webhook Trigger node]
- Payload Type: JSON
- Data:
  - title: `{{Title}}`
  - transcript: `{{Transcript}}`
  - action_items: `{{Summary}}` (Granola's enhanced summary includes action items)
  - attendees: `{{Attendees}}`
  - date: `{{Calendar Event Date}}`
  - link: `{{Link}}`

That's it for Zapier. All logic lives in N8N.

---

## N8N SIDE

### Workflow name: Jamie Work OS — Granola Capture

### Node 1 — Webhook Trigger
- HTTP Method: POST
- Path: granola-capture
- Respond: Immediately
- Copy the Production URL → paste into the Zapier POST step above

### Node 2 — Code in JavaScript (Claude extraction)
```javascript
const body = $input.first().json.body || $input.first().json;
const title = body.title || 'Untitled meeting';
const transcript = body.transcript || '';
const actionItems = body.action_items || '';
const attendees = body.attendees || '';
const date = body.date || new Date().toISOString().split('T')[0];
const today = new Date().toISOString().split('T')[0];

const systemPrompt = `You are processing a meeting transcript for Jamie Bawalan, Principal at BCG Manila. Return ONLY raw JSON, no markdown, no backticks.

Return an object with two fields:
1. "tasks": array of action items for Jamie, each {"tx":"task","pr":"PROJECT","ar":"case|bd|cit|pers","ur":"urgent|important|background","du":"YYYY-MM-DD or null"}
2. "summary": a 3-5 sentence markdown summary — key decisions, open questions, who owns what.

Projects: NAP/ADB/TFD/DRR/JGS/Meralco/PrimeEnergy/Aboitiz/ACEN/PrimeInfra/CDC/CDP/Recruiting/SOCCOM/Visa/Admin.
Contacts: Martin/Moad/Mark/Ali=ADB. VMC/DENR/PHILSURIN/James/Gina/Kitty/Maxine=NAP. Jaymes/Vidhya/Eugenia=TFD. Toni/Tiffany/Edgar=DRR. Lance=JGS. Donna/Razon=PrimeEnergy. Andrew=Meralco. Hannah/Joey/Veronica/Naomi/Jan/Gwen/William/Anika=CDC. John Soriano=CDP. Ritz/Franco/Ray=Recruiting.
Today is ${today}. Only extract genuine action items Jamie must do or follow up on. Ignore items fully owned by others.`;

const userMsg = 'Meeting: ' + title + '\nDate: ' + date + '\nAttendees: ' + attendees + '\n\nGranola summary + action items:\n' + actionItems + '\n\nFull transcript:\n' + transcript;

const response = await this.helpers.httpRequest({
  method: 'POST',
  url: 'https://api.anthropic.com/v1/messages',
  headers: {
    'anthropic-version': '2023-06-01',
    'content-type': 'application/json',
    'x-api-key': 'YOUR_ANTHROPIC_KEY'
  },
  body: {
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1500,
    system: systemPrompt,
    messages: [{ role: 'user', content: userMsg }]
  },
  json: true
});

const content = response.content?.[0]?.text || '{}';
return [{ json: { content, title, date } }];
```

### Node 3 — Code in JavaScript (Parse + Write to GitHub)
```javascript
let raw = $input.first().json;
let content = raw.content.replace(/```json|```/g,'').trim();
let parsed;
try { parsed = JSON.parse(content); } catch(e) { parsed = {tasks:[], summary:''}; }

const tasks = parsed.tasks || [];
const summary = parsed.summary || '';
const title = raw.title;
const date = raw.date;
const token = 'YOUR_GITHUB_TOKEN';
const repo = 'jamiebawalan/jamie-work-os';

// --- 1. Write tasks to pending.json ---
const getPending = await this.helpers.httpRequest({
  method: 'GET',
  url: `https://api.github.com/repos/${repo}/contents/pending.json`,
  headers: {'Authorization':`token ${token}`,'Accept':'application/vnd.github.v3+json'}
});
const pendingSha = getPending.sha;
const pending = JSON.parse(Buffer.from(getPending.content.replace(/\n/g,''),'base64').toString('utf8'));

const prefix = 'gr' + Date.now();
tasks.forEach((t,i) => {
  t.id = prefix + '_' + i;
  t.source = title;
  t.dn = 0;
  t.added_at = Date.now();
  pending.pending.push(t);
});
pending.last_updated = new Date().toISOString();

await this.helpers.httpRequest({
  method: 'PUT',
  url: `https://api.github.com/repos/${repo}/contents/pending.json`,
  headers: {'Authorization':`token ${token}`,'Accept':'application/vnd.github.v3+json','Content-Type':'application/json'},
  body: {
    message: 'Granola pending: ' + title,
    content: Buffer.from(JSON.stringify(pending, null, 2)).toString('base64'),
    sha: pendingSha
  },
  json: true
});

// --- 2. Write summary to meetings.json ---
const getMeet = await this.helpers.httpRequest({
  method: 'GET',
  url: `https://api.github.com/repos/${repo}/contents/meetings.json`,
  headers: {'Authorization':`token ${token}`,'Accept':'application/vnd.github.v3+json'}
});
const meetSha = getMeet.sha;
const meet = JSON.parse(Buffer.from(getMeet.content.replace(/\n/g,''),'base64').toString('utf8'));

meet.meetings.push({
  id: prefix,
  title: title,
  date: date,
  summary: summary,
  processed: false,
  added_at: Date.now()
});
meet.last_updated = new Date().toISOString();

await this.helpers.httpRequest({
  method: 'PUT',
  url: `https://api.github.com/repos/${repo}/contents/meetings.json`,
  headers: {'Authorization':`token ${token}`,'Accept':'application/vnd.github.v3+json','Content-Type':'application/json'},
  body: {
    message: 'Granola summary: ' + title,
    content: Buffer.from(JSON.stringify(meet, null, 2)).toString('base64'),
    sha: meetSha
  },
  json: true
});

return [{ json: { tasks_added: tasks.length, summary_saved: true, title } }];
```

### Node 4 — Respond to Webhook (optional)
- Returns 200 to Zapier so it knows the POST succeeded

---

## VAULT SYNC (Claude Desktop, manual or scheduled)
In a Claude Desktop vault conversation, say:
"Process new meetings from meetings.json into the account MD files."

Claude reads meetings.json, finds entries with processed:false, appends each summary to the relevant account MD file (e.g. NAP.md), and marks them processed. This keeps account files enriched after every meeting without N8N needing OneDrive access.

---

## data file schemas

### pending.json
```json
{"last_updated":"ISO","pending":[{"id":"gr..._0","tx":"","pr":"","ar":"","ur":"","du":null,"dn":0,"source":"meeting title","added_at":0}]}
```

### meetings.json
```json
{"last_updated":"ISO","meetings":[{"id":"gr...","title":"","date":"","summary":"markdown","processed":false,"added_at":0}]}
```

---

## App side (already built into index.html)
- **Pending tab** — shows tasks from pending.json grouped by meeting source; approve (✓) or reject (✕) each, or approve/reject all
- Approved tasks → task list (localStorage); rejected → discarded
- `processedPending` array in localStorage tracks handled items so they don't reappear
- Red badge on Pending tab shows count awaiting review
