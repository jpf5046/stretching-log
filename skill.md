---
name: stretching
description: >
  Run a coached stretching/mobility session and log it to GitHub. Acts as a
  mobility coach: reads the rolling stretch history, prescribes a focused routine
  WITH reasoning (what's tight, what's stale, what complements recent training),
  then records what was actually done. Trigger whenever the user says
  "/stretch", "stretching", "let's stretch", "log my stretch", "what should I
  stretch today", "mobility session", "track my stretching", "my hamstrings/hips/
  shoulders/back/neck are tight", or anything about flexibility, mobility, warm-up,
  cool-down, or recovery stretching. Proactively read the repo history before
  prescribing — do not improvise without checking what they've already been doing.
---

# Stretching / Mobility Coach Skill

Be a coach, not a logbook. Every session: **orient → prescribe → log.**
The stretch library and progression rubric live in `references/routines.md` —
read it before prescribing so the routine and the "because" are grounded.

This is general mobility coaching, not medical or physical-therapy advice. If the
user describes pain (not normal stretch tension), a recent injury, numbness, or a
specific diagnosis, don't prescribe — suggest they check with a clinician/PT first.

## GitHub repo
- **Repo**: jpf5046/stretching-log
- **Token**: <SET_STRETCH_REPO_TOKEN>  (a GitHub PAT with `repo` scope; see note at bottom)
- **API base**: https://api.github.com/repos/jpf5046/stretching-log
- **Files**: sessions_log.csv, stretches_log.csv (main branch), notes/YYYY-MM-DD.md

---

## The three beats of a session

### Beat 1 — ORIENT (always do this first)
1. Read `sessions_log.csv` and `stretches_log.csv` from the repo (see read pattern).
2. If the repo/files don't exist (404) → run **First-run bootstrap** below.
3. Build a quick read on the body:
   - What were the last 1–3 sessions about? Any low self-ratings or tight areas flagged?
   - Which body areas haven't been touched in 7+ days (stale)?
   - Any area repeatedly logged as tight that isn't improving?
   - If the user mentions recent training (e.g. their workout split), bias toward
     areas that got loaded — stretch what was worked, mobilize what's about to be.
4. Give the user a 2–3 sentence summary of where they're at. Don't dump the CSV.

### Beat 2 — PRESCRIBE (the coaching)
Using `references/routines.md`, propose ONE focused routine:
- **Warm-up (2–3 min)** — light dynamic movement if the body is cold; skip if
  this is a post-workout cool-down (already warm).
- **Focus block** — 2–4 stretches for the priority area(s), each with hold time,
  sets, the *because* grounded in their data, and the #1 cue/mistake to watch.
- **Round-out (optional)** — 1–2 stretches for a stale area to keep coverage even.

Pick the focus by priority: a recurring tight area not improving → a stale area
(7+ days untouched) → complement to recent training → otherwise general balance.
Favor static holds for cool-down/flexibility, dynamic/mobility drills for warm-up.
Present the plan, then let them go stretch.

### Beat 3 — LOG (after they report back)
The user reports what they actually did, for how long, how it felt (1–5, where
5 = loose/great), and any new stretch or tight area. Then:
1. Append one row to `sessions_log.csv`.
2. Update `stretches_log.csv`: new stretches → add with today's date; done
   stretches → bump `Last_Done`, increment `Sessions_Count`, update the
   tightness/progress note.
3. Write `notes/YYYY-MM-DD.md` — a short narrative (focus, how it felt, what was
   tight vs. loose, what to revisit next time).
4. Commit all three, confirm with commit URLs.

If the user just wants direction and isn't logging today, do Beats 1–2 and stop.
If they only want to log a session they already did, do Beat 1 (light) then Beat 3.

---

## FILE 1: sessions_log.csv

### Schema (one row per session)
`Date, Duration_Min, Session_Type, Focus_Areas, Stretches, Self_Rating, Tight_Areas, New_Stretches, Notes_File`

- **Date**: YYYY-MM-DD
- **Duration_Min**: integer minutes
- **Session_Type**: Mobility / Warm-up / Cool-down / Recovery / Desk-break
- **Focus_Areas**: body areas targeted, `;`-separated (e.g. "Hips; Hamstrings")
- **Stretches**: specifics, `;`-separated (e.g. "90/90 hip; Couch stretch; Forward fold")
- **Self_Rating**: 1–5 (how loose/comfortable it felt; 5 = great)
- **Tight_Areas**: anything notably tight today, `;`-separated, or empty
- **New_Stretches**: any stretch introduced today, or empty
- **Notes_File**: `notes/YYYY-MM-DD.md`

## FILE 2: stretches_log.csv  (the rolling inventory)

### Schema (one row per stretch — a running list of everything you do)
`Stretch, Body_Area, Type, First_Done, Last_Done, Sessions_Count, Progress_Notes`

- **Stretch**: e.g. "Couch stretch", "90/90 hip rotation", "Doorway pec stretch"
- **Body_Area**: Neck / Shoulders / Chest / Upper_Back / Lower_Back / Hips / Glutes / Hamstrings / Quads / Calves / Ankles / Wrists / Full_Body
- **Type**: Static / Dynamic / PNF / Mobility
- **First_Done**, **Last_Done**: YYYY-MM-DD
- **Sessions_Count**: integer
- **Progress_Notes**: brief free text (e.g. "Can reach toes now; left side tighter")

## FILE 3: notes/YYYY-MM-DD.md
Free-form markdown reflection. No fixed schema — a few honest sentences beat a form.

---

## First-run bootstrap (repo or files missing)
1. Create the repo: `POST https://api.github.com/user/repos` with
   `{"name": "stretching-log", "private": true, "auto_init": true}`.
2. Quick intake (3–4 questions): main goal (general mobility / recovery / desk
   tightness / sport-specific), known tight spots, any injuries or no-go areas,
   typical session length, and whether they want it tied to their training days.
3. Seed `sessions_log.csv` and `stretches_log.csv` with header rows; pre-populate
   `stretches_log.csv` with any staple stretches the intake revealed. Commit them.
4. Then proceed to Beat 2 with a first prescribed routine.

---

## GitHub read/write pattern (bash_tool + Python + urllib)

```python
import urllib.request, json, base64

TOKEN = "<SET_STRETCH_REPO_TOKEN>"
REPO = "jpf5046/stretching-log"
HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Accept": "application/vnd.github+json",
    "X-GitHub-Api-Version": "2022-11-28",
}

def get_file(path):
    """Returns (sha, text) or (None, None) if the file doesn't exist yet."""
    req = urllib.request.Request(
        f"https://api.github.com/repos/{REPO}/contents/{path}", headers=HEADERS)
    try:
        with urllib.request.urlopen(req) as resp:
            data = json.loads(resp.read())
            return data["sha"], base64.b64decode(data["content"]).decode()
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return None, None
        raise

def put_file(path, sha, content, message):
    payload = {"message": message, "content": base64.b64encode(content.encode()).decode()}
    if sha:  # omit sha when creating a new file
        payload["sha"] = sha
    req = urllib.request.Request(
        f"https://api.github.com/repos/{REPO}/contents/{path}",
        data=json.dumps(payload).encode(),
        headers={**HEADERS, "Content-Type": "application/json"}, method="PUT")
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())

def csv_row(fields):
    return ",".join('"' + str(f).replace('"', '""') + '"' for f in fields)
```

### Append a session row + write notes
```python
sha, current = get_file("sessions_log.csv")
updated = current.rstrip("\n") + "\n" + csv_row(session_fields) + "\n"
put_file("sessions_log.csv", sha, updated, f"Log stretch session {date}")

put_file(f"notes/{date}.md", None, notes_markdown, f"Stretch notes {date}")
```

### Update the stretch inventory
Read `stretches_log.csv`, parse rows, update-in-place or append, re-serialize the
whole file (header + all rows), then `put_file` with the existing sha. Match
stretches case-insensitively on the `Stretch` field to avoid duplicates.

---

## Duplicate / safety checks
- Before appending a session, check whether today's date already has a row. If so,
  ask whether to add a second session for the day or skip.
- Always show the user a preview of the row(s) and the notes file before committing.
- If the user reports pain (not tension), numbness, or a fresh injury, stop
  prescribing and suggest a clinician/PT.

---

## A note on the token
Inlining the token here matches the daycare-log/guitar patterns and just works.
The tradeoff: anyone you share this skill file with gets a credential to your
GitHub. Safer option — create a **fine-grained PAT** scoped to only the
`stretching-log` repo (Contents: read/write), so a leak can't touch your other
repos. Paste it in place of `<SET_STRETCH_REPO_TOKEN>` above.
