---
name: volunteer-thread-tracker
description: Set up or refresh a visual Thread Tracker for a volunteer group's messy email (school council, committee, club): Gmail label + plus-address tag, timeline of ideas, open/closed loops.
---

# Volunteer Thread Tracker

Volunteer email is asymmetric. People reply to some of the group but not all of it, reply from personal addresses, start side threads, forward things days late, and decide things on calls that never reach the thread. Nobody owns the inbox, so actions get lost.

This skill fixes that in three parts:

1. **A Gmail filing system** that catches the stray mail.
2. **A visual tracker**: a timeline of who said what, plus a board of open and closed loops (asks, decisions, deadlines), each linked back to the email it came from.
3. **A refresh routine** that pulls new mail, adds events, and opens or closes loops with evidence.

Two modes: **Setup** (new group or workstream) and **Refresh** (update an existing tracker). If the user names an existing tracker or says "refresh", go to Refresh.

---

## Ground rules (apply in both modes)

- **Paraphrase, never infer.** Every event and loop says only what someone wrote or said. If it isn't in the source, it doesn't go in. The footer says so.
- **Every item links to its source**: a Gmail message URL, call minutes, or a doc. No orphan claims.
- **A loop closes only with evidence**: who closed it, when, and how ("Closed Nov 20: office confirmed the link is allowed (Priya, after her call)").
- **Flag contradictions, don't resolve them.** If two emails disagree (an award email says one design and the council line-up says another), put it in the event's context and open a loop.
- **No sending or settings changes without a yes.** Read mail freely. Drafting is fine. Creating labels or filters, labelling threads, or sending anything needs the user's OK first.
- **Volunteers' details stay private.** The tracker is an artifact that starts private to the user. Only share it with the group if the user asks to.

---

## Mode 1: Setup

### Step 1: Gmail filing system

Ask only for what you can't find: the group's name, the user's sending address, and which address the group should reply to.

Propose this system and set up what the connector allows once the user says yes:

| Piece | What it is | Why |
|---|---|---|
| **Label** | `Community/<Group>` (nested under a parent label) | One place for everything about the group |
| **Council tag** | A plus-address, e.g. `you+group@yourdomain` | CC it on every reply. A filter labels anything sent to it, so answers file themselves |
| **Filter 1** | `to:you+group@yourdomain` → apply label, never send to spam | Catches anything sent or CC'd to the tag |
| **Filter 2** | `from:(member1 OR member2 OR …)` or a subject keyword → apply label | Catches replies that skip the tag |
| **Forward rule** | Mail that lands at a personal address (iCloud, old account) gets forwarded to the tag | Brings off-thread mail into the folder; the tracker notes it ("sent to your personal address, forwarded in Nov 2") |

The Gmail connector can create labels and label threads. Filters usually have to be made by the user in Gmail settings (gear icon → *Filters and blocked addresses*). Give them the exact filter text to paste. Plus addresses work on Gmail and Google Workspace. If their provider doesn't support them, use a dedicated group address or a label-only filter instead.

### Step 2: Pull and read the mail

- Search the label, plus related forwards and known members' direct mail: `label:<label-slug>`, then `from:` and `to:` searches for anything the label missed. Paginate with `pageToken` until there are no more results.
- Read every thread in full. Also take in any call minutes, docs, or plan pages the user points to. These become events too, linked to the doc.
- Note which mail arrived outside the label (personal address, direct to one person). That asymmetry goes in the event's `context`.

### Step 3: Choose 3–5 topic lanes

Pick lanes from the content, e.g. *The plan, Family fund, Merch, Band night, Logistics*. Use short plain names. Always keep a **Logistics/Admin** lane for housekeeping (wrong addresses, meeting times). Each lane gets a colour slot `c1`–`c6`.

### Step 4: Extract events

**One event per idea, ask, or decision, not one per email.** One email with three distinct points becomes three events with the same timestamp and URL.

Each event:

```json
{
  "id": "short-kebab-id",
  "t": "<lane key>",
  "date": "YYYY-MM-DD",
  "time": "3:23 pm",
  "who": "Full name (role)",
  "whoShort": "First name or 'You'",
  "idea": "Headline in their voice, under ~10 words",
  "points": ["2–4 paraphrased specifics: numbers, dates, names, asks"],
  "context": "1–2 sentences linking to earlier events, what it answers or overrides, where it arrived",
  "subject": "Email subject",
  "to": ["who a reply should go to"],
  "cc": ["optional"],
  "toLabel": "Human label, e.g. 'Full council (reply all)'",
  "url": "Gmail message URL or doc URL",
  "small": false,
  "isNew": false
}
```

- `small: true` for minor items (acks, housekeeping) so the timeline breathes.
- Gmail message URL: `https://mail.google.com/mail/?authuser=<user>#all/thread-f:<threadId>|msg-f:<messageId>`, using the decimal IDs.
- Sort events by date and time. The user's own sends count as events ("You").

### Step 5: Extract loops

A loop is anything someone has to do, answer, or decide.

```json
{
  "t": "<lane key>",
  "status": "open",
  "q": "The action as an instruction: 'Answer Dana: does merch need money up front?'",
  "src": "Who asked, when, and what's blocking it",
  "due": "Human deadline: 'Before the Nov 12 decision'",
  "due_date": "YYYY-MM-DD (optional)",
  "ev": "<id of the event it came from or was closed by>"
}
```

- Write `q` as the next action, not the topic.
- Keep closed loops. They render struck through below the open ones, so the group can see what's been settled and when.
- Put the most urgent item first ("NOW (Mon Nov 23 am)") when something is blocking everything else.

### Step 6: Build and publish

1. Write `threads.json` with the tracker config, events, and loops (schema below).
2. Write `index.html` from the template at the end of this file. Paste the same JSON in place of `/*SEED*/null` so the page works even if the fetch fails.
3. Publish both files as one artifact (`index.html` + `threads.json`). Title: `<Group> Thread Tracker` or `<Workstream> · Thread Tracker`.
4. Look at it before handing it over: the bubbles read cleanly, filters work, Reply opens a Gmail draft with the tag on CC, and there are no empty lanes.

### Splitting workstreams

When one thread spawns a big sub-project (e.g. "Merch: final design, PO, order to cash"), start a **second tracker** with its own lanes and a different `stream` ID. Carry over 1–3 anchoring events from the parent (the deadline or decision that started it) so the new timeline has context.

---

## Mode 2: Refresh

1. **Load the current tracker**: read the artifact's `threads.json` and note `generated_at`.
2. **Pull new mail since `generated_at`**: the label, the tag, member addresses, and any forwards. Also check whether the user mentions calls, docs, or their own edits since the last refresh.
3. **De-dupe** by message URL. Skip anything already present.
4. **Clear the old flags**: set `isNew: false` on every existing event, then add the new events with `isNew: true`.
5. **Walk every open loop.** For each one, ask: did any new event answer it, decide it, or make it moot?
   - Yes → `status: "closed"`, rewrite `src` as "Closed <date>: <what happened> (<who>)", and point `ev` at the closing event.
   - Changed but still open → update `src` and `due`.
6. **Open new loops** for every new ask, decision needed, or deadline. Check for broken timelines too: if a new fact moves a date (the office needs 3 working days), open a loop saying the plan breaks and what has to move.
7. **Update the header's key dates** if prices, deadlines, or go-live dates changed.
8. **Bump** `generated_at` and write `source.note` ("Last pull: Mon Nov 23, 9:20 am. 26 new messages, 1 call").
9. **Publish only `threads.json`** (patch mode). The page fetches it and re-renders if it's newer than the seed. Also update the inline seed now and then so the two don't drift far apart.
10. **Report back briefly**: how many new events, which loops closed, which opened, and the **one thing that matters most right now**.

Refresh can run on a schedule (e.g. weekday mornings). If it's unattended, don't send anything. Just publish and leave a summary.

---

## threads.json schema

```json
{
  "contract": "thread-tracker/1",
  "stream": "main",
  "generated_at": "2026-11-23T09:25:00-05:00",
  "tracker": {
    "title": "Thread tracker",
    "eyebrow": "School name · Fundraising breakout",
    "sub": "Every idea from the emails tagged Community/Group. Tap a bubble for who said it, the context, and a reply that lands back in the folder.",
    "me": "you@yourdomain",
    "tag": "you+group@yourdomain",
    "topics": [
      {"key": "plan", "name": "The plan", "slot": 1},
      {"key": "admin", "name": "Logistics", "slot": 6}
    ],
    "keydates": [{"label": "Orders close", "text": "Sun Nov 15"}],
    "selected": "optional-event-id"
  },
  "source": {"note": "What was pulled and when"},
  "events": [],
  "loops": []
}
```

---

## Visual spec (what the template renders)

- **Header**: eyebrow, big title, one-line purpose, a key-dates strip, an "Updated" timestamp, and lane filter chips. Tapping one chip isolates that lane; tapping it again resets.
- **Timeline**: scrolls sideways, newest on the right (auto-scrolled). Each event is a speech bubble in its lane colour, alternating above and below a dated axis. Dashed **"+N days"** gaps mark quiet stretches. **NEW** badges mark the latest refresh.
- **Detail panel**: the selected bubble's headline, who, when, subject, points, and context. Beside it, a **Reply** card with copyable addresses and a *Reply in Gmail* button. The button opens a compose draft with the council tag on CC, so the answer files itself.
- **Loop board**: cards with a coloured lane edge. Open loops come first. Closed loops are faded and struck through. Each card has "See where it came from", which jumps to the source bubble.
- Light and dark themes, reduced-motion support, mobile layout (the detail panel stacks below 820px).

---

## Template: index.html

Replace `/*SEED*/null` with the full threads.json. Change nothing else unless the user asks for a design change.

```html
<!doctype html>
<html lang="en"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<title>Thread Tracker</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,500;12..96,700&family=IBM+Plex+Sans:wght@400;500;600&family=IBM+Plex+Mono:wght@400;500&display=swap">
<style>
:root{
  --bg:#f3f5f8;--surface:#fff;--ink:#18202c;--muted:#5b6676;--line:#d6dce5;--focus:#2f4f8f;
  --c1:#2f4f8f;--c2:#0f7f74;--c3:#8a3f86;--c4:#b5651a;--c5:#a33a3a;--c6:#6b7486;
  --c1-bg:#e6ecf7;--c2-bg:#e1f2ef;--c3-bg:#f3e6f2;--c4-bg:#f8ecdf;--c5-bg:#f7e4e4;--c6-bg:#eceef2;
  --display:"Bricolage Grotesque","Avenir Next","Segoe UI",sans-serif;
  --body:"IBM Plex Sans","Segoe UI",system-ui,sans-serif;
  --mono:"IBM Plex Mono",ui-monospace,"SF Mono",Menlo,monospace;color-scheme:light}
@media (prefers-color-scheme:dark){:root{
  --bg:#12161d;--surface:#1b212b;--ink:#e7ebf1;--muted:#9aa5b5;--line:#2e3744;--focus:#8fb0f0;
  --c1:#8fb0f0;--c2:#5cc9bb;--c3:#d99ad4;--c4:#eaa45f;--c5:#ec8a8a;--c6:#a6afbf;
  --c1-bg:#1f2b42;--c2-bg:#15302d;--c3-bg:#30202f;--c4-bg:#33261a;--c5-bg:#3a1f1f;--c6-bg:#232a35;color-scheme:dark}}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:var(--ink);font:15px/1.55 var(--body)}
.s1{--tc:var(--c1);--tbg:var(--c1-bg)}.s2{--tc:var(--c2);--tbg:var(--c2-bg)}.s3{--tc:var(--c3);--tbg:var(--c3-bg)}
.s4{--tc:var(--c4);--tbg:var(--c4-bg)}.s5{--tc:var(--c5);--tbg:var(--c5-bg)}.s6{--tc:var(--c6);--tbg:var(--c6-bg)}
.wrap{max-width:1180px;margin:0 auto;padding:28px max(16px,env(safe-area-inset-left)) 56px;display:flex;flex-direction:column;gap:28px}
header{display:flex;flex-wrap:wrap;justify-content:space-between;align-items:flex-end;gap:16px}
.eyebrow,.label{font:12px var(--mono);letter-spacing:.08em;text-transform:uppercase;color:var(--muted)}
h1{font:700 clamp(28px,4.4vw,40px)/1.05 var(--display);margin:6px 0 0;text-wrap:balance}
.sub{color:var(--muted);margin:8px 0 0;max-width:60ch}
.keydates{display:flex;flex-wrap:wrap;gap:6px 14px;margin:10px 0 2px;font-size:13px;color:var(--muted)}.keydates b{color:var(--ink);font-weight:600}
.updated{font:12px var(--mono);color:var(--muted);margin-top:6px}
.legend{display:flex;flex-wrap:wrap;gap:8px}
.chip{font:500 13px var(--body);border:1px solid var(--line);background:var(--surface);color:var(--ink);border-radius:999px;padding:5px 12px 5px 10px;display:inline-flex;align-items:center;gap:7px;cursor:pointer}
.chip .sw{width:9px;height:9px;border-radius:50%;background:var(--tc)}.chip[aria-pressed=true]{border-color:var(--ink)}
.chip:focus-visible,.bubble:focus-visible,.btn:focus-visible{outline:2px solid var(--focus);outline-offset:2px}
.tl-shell{background:var(--surface);border:1px solid var(--line);border-radius:14px;overflow-x:auto}
.tl{display:flex;align-items:stretch;padding:22px 20px 14px;min-width:max-content}
.col{width:186px;display:flex;flex-direction:column;align-items:center;flex:none}
.gap{width:58px;flex:none;display:flex;flex-direction:column;justify-content:flex-end;align-items:center}
.gap span{font:11px var(--mono);color:var(--muted);background:var(--bg);border:1px dashed var(--line);border-radius:6px;padding:1px 6px;margin-bottom:21px;white-space:nowrap}
.gap i{width:100%;height:2px;background:repeating-linear-gradient(90deg,var(--line) 0 5px,transparent 5px 9px);margin-bottom:36px}
.slot{height:190px;display:flex;align-items:flex-end;width:100%;padding:0 6px}.col.hi .slot{align-items:flex-start}
.bubble{position:relative;width:100%;text-align:left;font:inherit;color:var(--ink);background:var(--tbg);border:1.5px solid var(--tc);border-radius:16px;padding:10px 12px 11px;cursor:pointer;transition:transform .15s,box-shadow .15s,opacity .2s}
.bubble::after{content:"";position:absolute;left:50%;bottom:-8px;width:14px;height:14px;background:var(--tbg);border-right:1.5px solid var(--tc);border-bottom:1.5px solid var(--tc);transform:translateX(-50%) rotate(45deg)}
.bubble:hover{transform:translateY(-2px)}.bubble[aria-pressed=true]{box-shadow:0 0 0 3px var(--tc)}
.bubble .who{font:11px var(--mono);color:var(--tc);text-transform:uppercase;letter-spacing:.05em;display:block;margin-bottom:3px}
.bubble .idea{font:500 15.5px/1.2 var(--display);display:block}.bubble.small .idea{font-size:14px}
.dim{opacity:.22}.stem{width:1.5px;flex:1;min-height:14px;background:var(--line)}
.axis{position:relative;width:100%;height:46px;display:flex;flex-direction:column;align-items:center}
.axis::before{content:"";position:absolute;top:8px;left:0;right:0;height:2px;background:var(--line)}
.dot{position:relative;width:14px;height:14px;border-radius:50%;background:var(--surface);border:3px solid var(--tc);margin-top:2px}
.date{font:12px var(--mono);color:var(--muted);margin-top:6px}
.hint{font-size:13px;color:var(--muted);margin-top:-16px}
.new{font:600 10px var(--mono);background:var(--tc);color:var(--surface);border-radius:4px;padding:1px 4px;margin-left:4px}
.detail{display:grid;grid-template-columns:minmax(0,1.6fr) minmax(0,1fr);gap:20px}@media (max-width:820px){.detail{grid-template-columns:1fr}}
.card{background:var(--surface);border:1px solid var(--line);border-radius:14px;padding:22px 24px}
.card h2{font:700 24px/1.15 var(--display);margin:8px 0 12px;text-wrap:balance}
.tag{display:inline-flex;font:500 12px var(--mono);text-transform:uppercase;letter-spacing:.06em;color:var(--tc);background:var(--tbg);border-radius:999px;padding:3px 10px}
.meta{display:flex;flex-wrap:wrap;gap:6px 16px;font-size:13px;color:var(--muted);margin-bottom:14px}
.points{margin:0 0 14px;padding-left:18px;display:flex;flex-direction:column;gap:6px}
.context{border-top:1px solid var(--line);padding-top:12px;font-size:14px;color:var(--muted)}.context b{color:var(--ink)}
.side{display:flex;flex-direction:column;gap:14px}.person{font:500 20px var(--display)}
.addr{display:flex;flex-direction:column;gap:6px}.addrrow{display:flex;gap:8px;align-items:center;min-width:0}
.addrrow code{font:13px var(--mono);background:var(--bg);border:1px solid var(--line);border-radius:8px;padding:6px 9px;flex:1;min-width:0;overflow-wrap:anywhere;user-select:all}
.btn{font:500 14px var(--body);display:inline-flex;align-items:center;gap:6px;border-radius:10px;padding:9px 14px;border:1px solid var(--line);background:var(--surface);color:var(--ink);text-decoration:none;cursor:pointer}
.btn.primary{background:var(--ink);color:var(--bg);border-color:var(--ink)}.btn.sm{padding:6px 10px;font-size:13px}
.actions{display:flex;flex-wrap:wrap;gap:8px}.note{font-size:12.5px;color:var(--muted)}
.loops h3{font:700 22px var(--display);margin:0 0 12px}
.loopgrid{display:grid;grid-template-columns:repeat(auto-fill,minmax(260px,1fr));gap:12px}
.loop{background:var(--surface);border:1px solid var(--line);border-left:4px solid var(--tc);border-radius:10px;padding:14px 16px;display:flex;flex-direction:column;gap:6px}
.loop .q{font-weight:600}.loop .src{font-size:13px;color:var(--muted)}.loop .due{font:12px var(--mono);color:var(--tc)}
.loop.closed{border-left-color:var(--line);opacity:.7}.loop.closed .q{text-decoration:line-through;text-decoration-color:var(--muted)}
.loop .state{font:500 11px var(--mono);text-transform:uppercase;letter-spacing:.06em;color:var(--muted)}
footer{font-size:12.5px;color:var(--muted)}
@media (prefers-reduced-motion:reduce){*{transition:none!important}}
</style></head><body>
<div class="wrap">
  <header><div>
    <div class="eyebrow" id="eyebrow"></div><h1 id="title"></h1><p class="sub" id="sub"></p>
    <div class="keydates" id="keydates"></div><div class="updated" id="updated"></div></div>
    <div class="legend" id="legend" role="group" aria-label="Filter by topic"></div></header>
  <div class="tl-shell" id="tlshell"><div class="tl" id="tl"></div></div>
  <div class="hint">Scroll sideways for the full run. Dashed gaps show quiet stretches.</div>
  <section class="detail" id="detail" aria-live="polite"></section>
  <section class="loops"><h3>Open loops</h3><div class="loopgrid" id="loops"></div></section>
  <footer><span id="srcline"></span> Items are paraphrased from the emails and minutes; nothing is inferred beyond what was written or said. Replies open a Gmail draft with the group tag on CC so the answer gets filed.</footer>
</div>
<script>
const SEED = /*SEED*/null;
let DATA = SEED, T = {}, active = new Set(), sel = null;
const esc = s => String(s ?? "").replace(/[&<>"']/g, c => ({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"}[c]));
const fmt = d => new Date(d+"T12:00:00").toLocaleDateString(undefined,{month:"short",day:"numeric"});
const stamp = iso => new Date(iso).toLocaleString(undefined,{month:"short",day:"numeric",hour:"numeric",minute:"2-digit"});
const days = (a,b) => Math.round((new Date(b)-new Date(a))/864e5);
const $ = id => document.getElementById(id);
const cls = k => "s" + ((T[k] && T[k].slot) || 6);

function composeUrl(e){
  const C = DATA.tracker, subj = /^re:/i.test(e.subject) ? e.subject : "Re: " + e.subject;
  const p = new URLSearchParams({view:"cm", fs:"1", to:(e.to||[]).join(","), cc:[C.tag].concat(e.cc||[]).filter(Boolean).join(","), su:subj});
  if (C.me) p.set("authuser", C.me);
  return "https://mail.google.com/mail/?" + p.toString();
}
function renderHeader(){
  const C = DATA.tracker;
  document.title = C.title || "Thread Tracker";
  $("eyebrow").textContent = C.eyebrow || ""; $("title").textContent = C.title || "Thread tracker"; $("sub").textContent = C.sub || "";
  $("keydates").innerHTML = (C.keydates||[]).map(k => `<span><b>${esc(k.label)}</b> ${esc(k.text)}</span>`).join("");
  $("updated").textContent = "Updated " + stamp(DATA.generated_at);
  $("srcline").textContent = DATA.source && DATA.source.note ? "Source: " + DATA.source.note : "";
}
function renderLegend(){
  $("legend").innerHTML = Object.entries(T).map(([k,v]) =>
    `<button class="chip ${cls(k)}" aria-pressed="${active.has(k)}" data-k="${k}"><span class="sw"></span>${esc(v.name)}</button>`).join("");
  $("legend").querySelectorAll(".chip").forEach(b => b.onclick = () => {
    const k = b.dataset.k, all = Object.keys(T);
    if (active.size === all.length) active = new Set([k]);
    else if (active.has(k) && active.size === 1) active = new Set(all);
    else active.has(k) ? active.delete(k) : active.add(k);
    renderLegend(); renderTimeline();
  });
}
function renderTimeline(){
  let html = "", prev = null;
  DATA.events.forEach((e,i) => {
    if (prev){ const g = days(prev.date, e.date); if (g >= 3) html += `<div class="gap"><span>+${g} days</span><i></i></div>`; }
    html += `<div class="col ${cls(e.t)}${i%2?" hi":""}"><div class="slot">
      <button class="bubble${e.small?" small":""}${active.has(e.t)?"":" dim"}" data-id="${esc(e.id)}" id="b-${esc(e.id)}" aria-pressed="${sel===e.id}">
      <span class="who">${esc(e.whoShort)}${e.isNew?' <b class="new">New</b>':""}</span><span class="idea">${esc(e.idea)}</span></button></div>
      <div class="stem"></div><div class="axis"><div class="dot"></div><div class="date">${fmt(e.date)}</div></div></div>`;
    prev = e;
  });
  $("tl").innerHTML = html;
  $("tl").querySelectorAll(".bubble").forEach(b => b.onclick = () => select(b.dataset.id, false));
}
function select(id, scroll){
  sel = id;
  document.querySelectorAll(".bubble").forEach(b => b.setAttribute("aria-pressed", b.dataset.id === id));
  renderDetail();
  if (scroll){
    const smooth = matchMedia("(prefers-reduced-motion: reduce)").matches ? "auto" : "smooth";
    const b = $("b-"+id); if (b) b.scrollIntoView({behavior:smooth, inline:"center", block:"nearest"});
    $("detail").scrollIntoView({behavior:smooth, block:"start"});
  }
}
function renderDetail(){
  const e = DATA.events.find(x => x.id === sel); if (!e) { $("detail").innerHTML = ""; return; }
  const to = e.to || [];
  const addrs = to.length > 3
    ? `<div class="addrrow"><code id="addr-all">${esc(to.join(", "))}</code><button class="btn sm" data-copy="addr-all">Copy</button></div>`
    : to.map((a,i) => `<div class="addrrow"><code id="addr-${i}">${esc(a)}</code><button class="btn sm" data-copy="addr-${i}">Copy</button></div>`).join("");
  const open = /docs\.google|drive\.google/.test(e.url) ? "Open doc" : /mail\.google/.test(e.url) ? "Open email" : "Open source";
  $("detail").innerHTML = `
    <article class="card ${cls(e.t)}"><span class="tag">${esc((T[e.t]||{}).name || e.t)}</span>
      <h2>${esc(e.idea)}</h2>
      <div class="meta"><span>${esc(e.who)}</span><span>${fmt(e.date)} · ${esc(e.time)}</span><span>“${esc(e.subject)}”</span></div>
      <ul class="points">${(e.points||[]).map(p => `<li>${esc(p)}</li>`).join("")}</ul>
      <div class="context"><b>Context.</b> ${esc(e.context)}</div></article>
    <aside class="card side ${cls(e.t)}">
      <div><div class="label">Reply to</div><div class="person">${esc(e.toLabel)}</div></div>
      <div class="addr">${addrs}</div>
      <div class="actions"><a class="btn primary" href="${esc(composeUrl(e))}" target="_blank" rel="noopener">Reply in Gmail ↗</a>
        ${e.url ? `<a class="btn" href="${esc(e.url)}" target="_blank" rel="noopener">${open} ↗</a>` : ""}</div>
      <div class="note">The draft CCs <span style="font-family:var(--mono)">${esc(DATA.tracker.tag)}</span> so it lands in the group folder.</div></aside>`;
  document.querySelectorAll("[data-copy]").forEach(b => b.onclick = async () => {
    const node = $(b.dataset.copy);
    try { await navigator.clipboard.writeText(node.textContent); b.textContent = "Copied"; }
    catch(_) { const r = document.createRange(); r.selectNodeContents(node); const s = getSelection(); s.removeAllRanges(); s.addRange(r); b.textContent = "Selected"; }
    setTimeout(() => b.textContent = "Copy", 1500);
  });
}
function renderLoops(){
  const ord = DATA.loops.map((l,i) => [l,i]).sort((x,y) => (x[0].status==="closed") - (y[0].status==="closed"));
  $("loops").innerHTML = ord.map(([l,i]) => `
    <div class="loop ${cls(l.t)}${l.status==="closed"?" closed":""}">
      ${l.status==="closed" ? '<div class="state">Closed</div>' : ""}
      <div class="q">${esc(l.q)}</div><div class="src">${esc(l.src)}</div>
      ${l.due ? `<div class="due">${esc(l.due)}</div>` : ""}
      ${l.ev ? `<div><button class="btn sm" data-ev="${esc(l.ev)}">See where it came from</button></div>` : ""}</div>`).join("");
  document.querySelectorAll("[data-ev]").forEach(b => b.onclick = () => select(b.dataset.ev, true));
}
function renderAll(){
  T = Object.fromEntries((DATA.tracker.topics||[]).map(t => [t.key, t]));
  active = new Set(Object.keys(T));
  if (!sel || !DATA.events.find(x => x.id === sel)) sel = DATA.tracker.selected && DATA.events.find(x => x.id === DATA.tracker.selected) ? DATA.tracker.selected : (DATA.events.length ? DATA.events[DATA.events.length-1].id : null);
  renderHeader(); renderLegend(); renderTimeline(); renderDetail(); renderLoops();
  requestAnimationFrame(() => { const s = $("tlshell"); s.scrollLeft = s.scrollWidth; });
}
if (DATA) renderAll();
fetch("threads.json", {cache:"no-store"}).then(r => r.ok ? r.json() : null).then(d => {
  if (!d || d.contract !== "thread-tracker/1" || !Array.isArray(d.events)) return;
  if (DATA && (d.stream||"main") !== (DATA.stream||"main")) return;
  if (DATA && new Date(d.generated_at) <= new Date(DATA.generated_at)) return;
  DATA = d; renderAll();
}).catch(() => {});
</script>
</body></html>
```
