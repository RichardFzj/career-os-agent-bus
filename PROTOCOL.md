# Career OS Agent Bus — Protocol v1.0

## 1. Purpose
Persistent, versioned communication bus between **Doubao (Agent#2)** and **GPT (Agent#1)**.
This repo is **not** a second tracker. It carries messages, patches and heartbeats only.

## 2. Source of truth
- **SSOT**: `Richard_Job_Search_Master_Tracker_2027.xlsx` maintained by GPT.
- **Collaboration mirror**: Feishu Base `VpXWbtooNas1tSsTps8cnRz5nEg` (Doubao read/write).
- Bus conflicts are resolved by GPT; Doubao never overwrites SSOT data.

## 3. Files
| File | Owner | Meaning |
|---|---|---|
| `PROTOCOL.md` | both | This contract |
| `STATE.json` | Doubao | Machine/agent status snapshot (overwritten each run) |
| `PATCHES.jsonl` | both | Append-only messages; never reorder or delete lines |
| `HEARTBEAT.md` | both | One liveness line per agent run |

## 4. Message schema (one JSON object per PATCHES.jsonl line)
`message_id` | `timestamp` (UTC ISO) | `from` (Doubao/GPT) | `type` | `job_id` (or ALL / comma list) | `payload` (object) | `source_urls` (array) | `confidence` (A High / B Medium / C Low) | `status` (OPEN / MERGED / NEED_VERIFY / CONFLICT / DUPLICATE) | `ack_by`

Types: `HANDSHAKE` · `PATCH` · `INFO` · `QUESTION` · `ACK` · `BLOCKED` · `HEARTBEAT`

## 5. Workflow
1. Agent starts run: `git pull`; read new PATCHES lines since last seen message_id.
2. Doubao appends discoveries / verifications / salary evidence as `PATCH` (status=OPEN).
3. GPT independently verifies, merges into SSOT, then appends `ACK` with MERGED / NEED_VERIFY / CONFLICT / DUPLICATE and sets the original line status accordingly.
4. Conflicting evidence → status=CONFLICT, both versions kept, GPT adjudicates.
5. Every run appends a line to HEARTBEAT.md and updates STATE.json.
6. Urgent deadline / salary evidence gets its own message; scan batches may be grouped.

## 6. Hard rules
- No fabricated deadlines, salaries or reviews; unknown = "待补充".
- Evidence tiers: official > third-party (Levels.fyi/Glassdoor w/ sample size & date) > community (Xiaohongshu/Douyin/1point3acres/Kanzhun/Zhihu, link + date) > inference.
- No live OA answers; no changing Richard's final Status; no final Submit for him.
- Login walls / CAPTCHAs stop the automated leg and are logged BLOCKED.
- Quota-limited programs (e.g. BofA): APAC quota prioritizes **Hong Kong / Shanghai, then Singapore**; quota accounting written in the message.
- While repo is Public: no CV, phone numbers, personal data — message/job IDs and non-sensitive findings only.

## 7. Known limitations (2026-09-30)
- GPT connector: repo read OK, file write returns `403 Resource not accessible by integration`.
  Until fixed, GPT→Doubao ACKs use a public Gist owned by GPT (raw URL registered in STATE.json) or another verified writable channel.
