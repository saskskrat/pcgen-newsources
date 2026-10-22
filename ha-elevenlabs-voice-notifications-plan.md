# PROJECT BRIEF: Cloned-Voice Notifications on Google Nest Speakers (Home Assistant OS + ElevenLabs)

> **Audience:** an LLM agent (target model: Claude Sonnet 5.5) with access to a Home Assistant instance through the HA-MCP toolset.
> **Read this entire file before taking any action.** Follow phases in order. Do not skip checkpoints.

---

## 0. Operating rules (non-negotiable)

1. **Verify, never assume.** Every entity ID, URL, path, service name, and integration option in this document marked `⟨VERIFY⟩` or written as `<placeholder>` must be confirmed against the live system (or by asking the human) before use. If a fact in this brief contradicts the live system, the live system wins; report the discrepancy.
2. **Ask before you guess.** The human prefers clarifying questions over assumptions. If a required value is missing, stop and ask. Batch your questions (max 4 per message) and give a recommended default for each.
3. **Read-only first.** Phase 1 is discovery only. No writes until the human approves the Phase 1 report.
4. **Back up before changing.** Before the first write (Phase 3 onward), create a Home Assistant backup using `ha_manage_backup` and confirm it succeeded. Tell the human the backup name.
5. **Follow HA best practices.** At session start, load the HA-MCP skill `home-assistant-best-practices` (resource `skill://home-assistant-best-practices/SKILL.md`, or the `ha_get_skill_guide` fallback) and read the reference files relevant to each task (automations, scripts, helpers). Key rules: use `entity_id` not `device_id`; use native triggers/conditions over Jinja templates where possible; choose the correct automation `mode`; never edit `.storage` directly; do not instruct the user to hand-edit `configuration.yaml` for things that can be configured in the UI.
6. **Secrets.** The ElevenLabs API key must never be printed in chat, written into automations, scripts, or committed files. It lives only in the integration's config entry (UI) and, if Phase 6 is used, in `secrets.yaml`.
7. **Minimal footprint.** Create only the entities/files listed in this plan. Prefix everything you create with `vn_` (voice notifications) or use the `Voice Notify` label/category so it is easy to find and remove.
8. **Report faithfully.** If a test fails, say so with the actual output. Do not claim success without the verification in §9.
9. **Stop conditions.** Stop and ask the human if: a backup fails; an expected entity doesn't exist; you'd need to modify an existing automation; a step requires credentials you don't have; or anything would affect something outside this scope.

---

## 1. Known facts (provided by the human)

| Item | Value |
|---|---|
| Voice provider | ElevenLabs, **Starter plan** (just purchased) |
| Voice type | **Instant Voice Clone (IVC)**, ~3 minutes of source speech |
| Home Assistant install | **HAOS** (Home Assistant Operating System) |
| Playback devices | **Google Nest speakers** (Cast protocol) |
| Initial notification scope | **People arriving home / leaving home** only. More will be added later. |
| Existing notifications | Essentially none yet |
| Preferred approach | Pre-recorded clips of the cloned voice (cheap, offline-safe), with optional live TTS later |

## 2. Unknowns that MUST be resolved (see Phase 1 and §10)

- Which `person.*` entities exist, and their display names.
- Entity IDs of the Nest speakers / speaker groups (`media_player.*`).
- Whether the official ElevenLabs integration is already installed and which voices it lists.
- HA internal URL (must be reachable by the Nest speakers, plain HTTP is safest).
- Whether the Nest speakers should announce at night, and at what volume.
- Whether the human wants departure announcements, arrival announcements, or both, and whether guests/visitors are ever home alone.

---

## 3. Goal and non-goals

**Goal:** When a household member arrives home or leaves, a Nest speaker plays a short pre-generated MP3 spoken in the user's cloned ElevenLabs voice (e.g. "Welcome home, Sam."). Clips are stored locally on HA, so playback costs **zero** ElevenLabs credits after generation and still works if the subscription lapses.

**Non-goals (do not do these unless the human asks):**
- Exporting or backing up the cloned voice model itself (**not possible**; the model lives only on ElevenLabs servers).
- Building a conversational/voice-assistant pipeline.
- Changing existing automations, dashboards, or presence-detection configuration.
- Announcing on devices other than the approved Nest speakers.

---

## 4. Key technical facts and design decisions

### 4.1 The voice cannot be exported
ElevenLabs does not provide a download of cloned-voice model weights. What can be kept: **generated audio files**. The plan therefore stores MP3 output locally. Record the voice's `voice_id` (from ElevenLabs → Voices) in the project notes so clips can be regenerated later with the same voice while the account exists.

### 4.2 Two audio paths
| Path | How | Credit cost | Use for |
|---|---|---|---|
| **A. Pre-recorded clips (primary)** | MP3s generated once, stored in `/media/voice/`, played with `media_player.play_media` | One-time, tiny | Fixed phrases: arrive/leave, per person |
| **B. Live TTS (optional, Phase 6)** | `tts.speak` using the ElevenLabs TTS entity | Per message | Dynamic text (e.g. "Front door open for 5 minutes") |

Path A is implemented first. Path B is optional and must not be built without human approval.

### 4.3 Cast / Nest playback constraints ⟨VERIFY⟩
- Nest speakers fetch the audio URL themselves. The URL must be reachable from the speaker's network segment and should be **plain HTTP on the local network** (Cast devices commonly reject self-signed or untrusted HTTPS).
- Use a `media-source://` ID so HA resolves a correct URL. HAOS maps `/media` to the media source `media_source/local`. A file at `/media/voice/arrive_sam_1.mp3` is addressed as `media-source://media_source/local/voice/arrive_sam_1.mp3`.
- HA's **Internal URL** (Settings → System → Network) must be set to the LAN address, e.g. `http://<ha-lan-ip>:8123`. Confirm it; do not change it without approval.
- `media_player.play_media` with `announce: true` is intended to play over current media and resume afterward. Support varies by integration; **test it** (§9) and report actual behavior.
- Volume is not restored automatically after changing it. Baseline plan does **not** change volume. Per-time-of-day volume is an optional enhancement (§8, step 5).

### 4.4 Presence logic
- Use `person.*` entities (they combine device trackers) as triggers: `not_home → home` for arrival, `home → not_home` for departure.
- Debounce flapping GPS/Wi-Fi with a `for:` duration on the trigger (suggested: arrival `00:00:20`, departure `00:02:00`). ⟨Confirm with human⟩.
- Automation `mode: queued` (or `parallel` if two people arriving together should both be announced) so quick successive events aren't dropped. Ask the human which they prefer.

### 4.5 Clip catalogue design
- Filename pattern: `<event>_<personslug>_<n>.mp3`, lowercase, no spaces. Examples: `arrive_sam_1.mp3`, `arrive_sam_2.mp3`, `leave_sam_1.mp3`.
- Provide **2–3 variants per event per person** so it doesn't sound repetitive; the automation picks one at random.
- Add one generic fallback pair: `arrive_generic_1.mp3`, `leave_generic_1.mp3` ("Welcome home." / "Goodbye.") for persons with no clips.
- Estimated credit use: ~12–20 short clips ≈ a few hundred characters total, negligible against the Starter monthly allowance. ⟨VERIFY current credit-per-character cost and monthly allowance on the ElevenLabs pricing page before quoting numbers to the human.⟩

---

## 5. Architecture overview

```
ElevenLabs (voice_id, IVC)
        │  (one-time generation: web UI, or API script)
        ▼
  MP3 files  ──►  HA /media/voice/*.mp3      (kept forever, offline-safe)
                          │
person.<name> state change│
        ▼                 ▼
 automation vn_arrive / vn_leave
        │  picks clip name (random variant, fallback to generic)
        ▼
 script.vn_announce_clip(clip, targets)
        │  media_player.play_media (announce) via media-source://
        ▼
   Google Nest speaker(s)
```

Entities to be created (final names to be confirmed in Phase 2):

| Type | ID | Purpose |
|---|---|---|
| Script | `script.vn_announce_clip` | Plays one clip on the target speaker(s) |
| Automation | `automation.vn_arrival_announce` | Arrival announcements |
| Automation | `automation.vn_departure_announce` | Departure announcements |
| Input boolean | `input_boolean.vn_enabled` | Master on/off switch |
| Input select / group | speaker target (optional) | Choose which speaker(s) announce |

---

## 6. PHASE 1 — Discovery (READ-ONLY)

Use `ha_get_overview`, `ha_search`, `ha_search_tools` (to find further read tools), and `ha_call_read_tool`. **Do not write anything.**

Collect and report in a table:

1. **People:** all `person.*` entities, their friendly names, current state, and which trackers they use.
2. **Speakers:** all `media_player.*` entities with Cast platform; identify Nest speakers and any Cast groups. Report each one's current state and supported features if available.
3. **Integrations:** whether `elevenlabs` and `cast` integrations are loaded; whether `tts` entities exist (`tts.*`).
4. **Existing automations/scripts** that already mention person/arrival/departure/announce, to avoid duplication or conflict. Do not modify them.
5. **Config facts:** internal URL setting, whether media source `local` exists and `/media` is available (look for `media_source` and any `media-source://media_source/local` browsing results via available tools). If you cannot determine something with your tools, mark it **UNKNOWN** and put it in the questions list.
6. **Helpers:** any existing helpers beginning with `vn_` (should be none).

**Deliverable: "Discovery Report"** containing the findings plus a numbered question list (max 4 per batch) drawn from §10 that remain unanswered.

> ✋ **CHECKPOINT 1:** Present the Discovery Report. Wait for human answers and explicit approval to proceed.

---

## 7. PHASE 2 — Design confirmation

Using the human's answers, produce a short design for approval:

- Final speaker targets (entity IDs).
- Persons included and the spoken name for each (e.g. nicknames, pronunciation).
- Event list: arrival only, departure only, or both.
- Quiet hours (suggested default: no announcements 22:00–07:00 ⟨ask⟩) and whether to queue or drop them.
- Debounce values and automation mode.
- The **clip script** (exact text of every phrase, with variants). Draft the lines, keep each under ~8 words, and have the human approve or edit them.
- Whether to add a master `input_boolean.vn_enabled`.

Example draft table to present:

| File | Text |
|---|---|
| `arrive_<person>_1.mp3` | "Welcome home, <Name>." |
| `arrive_<person>_2.mp3` | "<Name>'s home." |
| `arrive_<person>_3.mp3` | "Hey <Name>, good to see you." |
| `leave_<person>_1.mp3` | "Bye, <Name>." |
| `leave_<person>_2.mp3` | "<Name> has left the house." |
| `arrive_generic_1.mp3` | "Welcome home." |
| `leave_generic_1.mp3` | "Goodbye." |

> ✋ **CHECKPOINT 2:** Human approves the design and phrase list.

---

## 8. PHASE 3 — Build

### Step 1 — Backup
Run `ha_manage_backup` to create a backup named `pre-voice-notifications-<date>`. Verify completion. Stop if it fails.

### Step 2 — ElevenLabs integration (only needed for Path B or to confirm the voice is visible)
The official **ElevenLabs** integration is added through Settings → Devices & Services → Add Integration → ElevenLabs (requires the API key and lets you choose a voice/model). ⟨VERIFY the integration name and that cloned voices appear in the voice list.⟩
- **Do not ask the human to paste the API key into chat.** Instruct them to enter it in the HA UI themselves.
- This step is optional for Path A-only. Ask the human whether to do it now or defer to Phase 6.

### Step 3 — Generate and place the clips (human-assisted; the agent cannot upload audio through MCP)
Provide the human with this procedure:

**Option 1: ElevenLabs web UI (recommended for a first batch)**
1. Open Text to Speech, select the cloned voice.
2. Choose the model and settings (suggested: keep the same model and settings for every clip so the voice sounds consistent; stability around 0.4–0.6 for natural variation ⟨these are suggestions, not requirements⟩).
3. Generate each approved line and download as MP3.
4. Rename files exactly to the catalogue names from Phase 2.

**Option 2: API script run on the human's own computer (not on HA)**
Provide a script such as the following; the human supplies the key via environment variable. ⟨VERIFY endpoint and parameters against current ElevenLabs API docs before handing this over.⟩

```bash
#!/usr/bin/env bash
# usage: ELEVEN_API_KEY=... VOICE_ID=... ./gen.sh "arrive_sam_1" "Welcome home, Sam."
set -euo pipefail
curl -sS -X POST \
  "https://api.elevenlabs.io/v1/text-to-speech/${VOICE_ID}?output_format=mp3_44100_128" \
  -H "xi-api-key: ${ELEVEN_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{\"text\": \"$2\", \"model_id\": \"eleven_multilingual_v2\"}" \
  --output "$1.mp3"
```

**Placing files on HAOS (pick one the human can do):**
- Settings → Media → *My media* → upload into a `voice` folder, **or**
- Samba share add-on → `media/voice/`, **or**
- File editor / SSH add-on → `/media/voice/`.

Ask the human to confirm when uploaded; then verify (browse media source via available tools if possible, otherwise rely on the playback test in §9).

### Step 4 — Create the playback script
Create `script.vn_announce_clip` (via the HA-MCP tools used for scripts/automations, or provide YAML for the human to paste into the UI if no write tool exists). ⟨VERIFY syntax against the user's HA version; current syntax uses `action:`.⟩

```yaml
alias: "VN: Announce clip"
description: "Plays a pre-generated voice clip on the Nest speaker(s). Part of the voice notifications project."
mode: queued
max: 5
fields:
  clip:
    name: Clip name (no extension)
    example: arrive_sam_1
    required: true
    selector:
      text:
  targets:
    name: Target media players
    required: true
    selector:
      entity:
        domain: media_player
        multiple: true
sequence:
  - condition: state
    entity_id: input_boolean.vn_enabled   # remove if not using the master switch
    state: "on"
  - action: media_player.play_media
    target:
      entity_id: "{{ targets }}"
    data:
      media_content_type: music
      media_content_id: "media-source://media_source/local/voice/{{ clip }}.mp3"
      announce: true
```

If the human declines the master switch, remove the first condition. If helpers are needed (`input_boolean.vn_enabled`), create them first.

### Step 5 — Create the presence automations
Use `ha_config_set_automation`. Replace placeholders with confirmed values. The example assumes variants `1..3` exist for each person; adjust to the real catalogue. Use a lookup so persons with no clips fall back to the generic clip.

**Arrival:**
```yaml
alias: "VN: Arrival announce"
mode: queued
max: 5
triggers:
  - trigger: state
    entity_id:
      - person.<person_a>
      - person.<person_b>
    from: not_home
    to: home
    for: "00:00:20"
conditions:
  - condition: time
    after: "07:00:00"
    before: "22:00:00"          # quiet hours, from Phase 2
actions:
  - variables:
      slug: "{{ trigger.to_state.object_id }}"
      known: ["<person_a>", "<person_b>"]
      clip: >-
        {% if slug in known %}arrive_{{ slug }}_{{ range(1, 4) | random }}
        {% else %}arrive_generic_1{% endif %}
  - action: script.vn_announce_clip
    data:
      clip: "{{ clip | trim }}"
      targets:
        - media_player.<nest_speaker>
```

**Departure:** same structure, `from: home`, `to: not_home`, `for: "00:02:00"`, clip prefix `leave_`, generic `leave_generic_1`. Variant range must match the number of files (e.g. `range(1, 3)` for two variants).

> The Jinja here is acceptable because no native option exists for random selection and slug-based lookup. Keep it minimal.

### Step 6 (optional) — Time-of-day volume
Only if the human asks. Snapshot the speaker's `volume_level`, set a lower level at night, play, then restore after the clip. Needs clip duration or a `wait_template` on the player state. Present the design to the human first.

> ✋ **CHECKPOINT 3:** Show the final YAML/entity list before enabling. Enable only after approval.

---

## 9. PHASE 4 — Verification (required before declaring success)

Run each test, record the result, and report them in a table with pass/fail and notes.

| # | Test | Expected |
|---|---|---|
| 1 | Call `script.vn_announce_clip` manually with `clip: arrive_generic_1`, one speaker | Clip plays audibly |
| 2 | Same call while the speaker is playing music (ask the human to start something) | Announcement plays, music resumes (note actual behavior; if it doesn't resume, tell the human) |
| 3 | Missing-file test: call with a non-existent clip name | Fails gracefully; report the error; confirm it doesn't break other automations |
| 4 | Simulate arrival: use Developer Tools → Actions to trigger the automation (`automation.trigger` with `skip_condition: true`) or have a person actually arrive | Correct person-specific clip plays |
| 5 | Departure simulation | Correct clip plays |
| 6 | Quiet-hours condition | Does not play outside the allowed window |
| 7 | Two near-simultaneous events | Both handled per chosen `mode`; none lost unexpectedly |
| 8 | `input_boolean.vn_enabled` off | Nothing plays |
| 9 | Review HA logs for errors from the new script/automations | No errors |

If playback fails, diagnose in this order: file path/name → media source ID → HA internal URL/HTTP reachability from the speaker → speaker online/idle state → `announce` support. Report findings; do not change network or URL settings without approval.

---

## 10. Questions to ask the human (batch; give a recommended default)

Batch 1 (after Discovery):
1. Which people should be announced, and how should each name be spoken? (default: the `person.*` friendly names)
2. Which Nest speaker(s) or group should announce? (default: the main living-area speaker only)
3. Arrival, departure, or both? (default: both)
4. Quiet hours? (default: no announcements 22:00–07:00)

Batch 2 (after Phase 2 draft):
5. Approve or edit the phrase list.
6. If two people arrive together, announce both (`parallel`/`queued`) or only the first? (default: both, queued)
7. Is it OK to add the `vn_enabled` master switch helper? (default: yes)
8. Do you want live TTS (Path B) for dynamic messages later? (default: not now)

---

## 11. PHASE 5 — Handoff

Provide the human with:
- A list of everything created (entity IDs, files, backup name).
- The clip catalogue (filename → text → `voice_id`/model/settings used) so the clips can be regenerated or extended.
- How to add a new person (generate 2–3 arrival + 2–3 departure clips, upload, add the person to the trigger list and the `known` list).
- How to add a new notification type (new filename prefix, new automation calling `script.vn_announce_clip`).
- A reminder: the clips remain usable even if the ElevenLabs subscription ends; the voice clone itself does not.

## 12. PHASE 6 (OPTIONAL) — Live TTS for dynamic messages

Only after explicit approval:
1. Add the ElevenLabs integration (human enters the API key in the UI).
2. Confirm the TTS entity ID (`tts.*`) and that the cloned voice is selectable. ⟨VERIFY⟩
3. Create a script wrapper, e.g. `script.vn_speak_live` with a `message` field, calling `tts.speak` against the target Nest speaker(s) with `announce`/caching options as supported. ⟨VERIFY parameters against current HA docs for the user's version.⟩
4. Warn the human that each new unique message consumes credits; recommend keeping live TTS to genuinely dynamic text and using Path A for anything repeated.
5. Set a character budget guideline and ask whether the human wants an alert near the monthly limit.

## 13. Rollback

To remove the project: delete `automation.vn_arrival_announce`, `automation.vn_departure_announce`, `script.vn_announce_clip`, any `vn_` helpers, and optionally the `/media/voice/` folder. Alternatively restore the backup created in Phase 3 Step 1. Always ask before deleting anything.

## 14. Risks and considerations to mention to the human

- **Privacy/security:** Arrival/departure announcements are audible to visitors and reveal who is home. Consider quiet hours and the `vn_enabled` switch.
- **Voice rights:** Only voices the user has the right to clone should be used; the Starter plan includes commercial usage terms ⟨VERIFY current terms⟩.
- **Clone quality:** IVC from ~3 minutes of speech may sound slightly inconsistent between generations; generating a few takes and keeping the best is recommended.
- **Presence accuracy:** GPS/Wi-Fi flapping can cause false announcements; the `for:` debounce mitigates this.
- **Dependency risk:** If the ElevenLabs account lapses, existing clips still play; regeneration may not be possible.

---

## 15. Definition of done

- [ ] Backup created and confirmed
- [ ] Clips uploaded to `/media/voice/` with the approved names
- [ ] `script.vn_announce_clip` created and tested on target speaker(s)
- [ ] Arrival and departure automations created, tested, and enabled
- [ ] All §9 tests recorded with results
- [ ] Handoff documentation delivered to the human
- [ ] No API keys or secrets exposed in chat, automations, or files
