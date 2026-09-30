# Agent Bus Protocol

Repository: RichardFzj/career-os-agent-bus; branch: main.

## Scope and source of truth

GitHub is transport/log only. The canonical single source of truth (SSOT) remains:
`/Job Search/Richard_Job_Search_Master_Tracker_2027.xlsx`

Messages are coordination events or proposed operations, not authoritative tracker records. An ACK confirms receipt only; it never proves that the tracker was updated. Only report an SSOT change after actually applying and verifying it through authorized access.

Never commit credentials, tokens, private keys, CV contents, or sensitive personal data, including in payloads, errors, commit messages, or attachments. Use opaque job IDs and minimal non-sensitive metadata. Keep the existing README unchanged.

## Message envelope

Every line of PATCHES.jsonl is one complete UTF-8 JSON object with exactly these fields, in this order:

message_id | timestamp | from | type | job_id | payload | status | ack_by

| Field | Value |
| --- | --- |
| message_id | Globally unique string, preferably UUID; stable across retries. |
| timestamp | Actual event time in UTC, ISO 8601 with Z suffix. |
| from | ChatGPT or Doubao. |
| type | INIT, HANDSHAKE, ACK, HEARTBEAT, PATCH, or ERROR. |
| job_id | Opaque job identifier, or null for bus events. |
| payload | JSON object containing minimal non-sensitive event data. |
| status | INFO, PENDING, ACKNOWLEDGED, or ERROR. |
| ack_by | null until acknowledgement; acknowledging agent name on an ACK event. |

PATCHES.jsonl is append-only: one JSON object per line, newline terminated, no blank lines, comments, arrays wrapping the log, or Markdown fences. Never edit, delete, reorder, or reformat existing lines. Corrections and acknowledgements are new events referencing the original message_id in payload.in_reply_to. Historical status and ack_by fields remain unchanged; effective status is derived from later events.

## Initial state

ChatGPT=ONLINE; Doubao=AWAITING_HANDSHAKE; round-trip=NOT_YET_VERIFIED.
ONLINE records availability at initialization, not a claim of a continuously running worker. STATE.json is a derived coordination snapshot, not the SSOT. HEARTBEAT.md explains liveness and verification.

## Handshake and ACK

1. Doubao uses its already-configured write-enabled deploy key to clone or pull main over SSH: git@github.com:RichardFzj/career-os-agent-bus.git. Read all four bus files.
2. Append one HANDSHAKE event with a new message_id, current UTC timestamp, from="Doubao", job_id=null, payload={"protocol_version":"1.0","ready":true}, status="PENDING", ack_by=null. Commit and push main, then read the pushed event back.
3. ChatGPT reads that remote event and appends an ACK with a new message_id, current UTC timestamp, from="ChatGPT", type="ACK", job_id=null, payload={"in_reply_to":"<Doubao HANDSHAKE message_id>"}, status="ACKNOWLEDGED", ack_by="ChatGPT". It commits and pushes the ACK.
4. Doubao pulls until it observes that matching ChatGPT ACK. It appends one receipt ACK with from="Doubao", payload={"in_reply_to":"<ChatGPT ACK message_id>","handshake_message_id":"<original HANDSHAKE message_id>","round_trip_received":true}, status="ACKNOWLEDGED", ack_by="Doubao", and pushes it.
5. ChatGPT reads the receipt from remote main before updating STATE.json to Doubao=ONLINE and round_trip=VERIFIED, with the three evidence IDs. Receipt ACKs must not trigger further ACKs.

Until step 5, retain round_trip=NOT_YET_VERIFIED. A push alone does not verify a round trip. Do not fabricate messages or acknowledgements for another agent.

## Safe writes, retries, and polling

Pull the latest main before writing. Deduplicate by message_id. On a rejected push, fetch latest main and reapply only missing new events after its complete existing log; preserve every remote line and never force-push. Validate JSON and unique IDs before pushing. Commit only intended bus files.

Wait for a matching ACK by periodically pulling while the agent's execution environment permits. If execution stops or times out, report AWAITING_ACK and resume later; never claim success or continuous background monitoring. Do not ask Richard to repeat GitHub or deploy-key setup unless an actual authorization failure makes it unavoidable.

HEARTBEAT events use current UTC timestamps and status="INFO"; they require no ACK. Liveness must be based on observed events, not merely the ONLINE snapshot.

## Compatibility and transport

Existing pre-initialization log records (including HANDSHAKE-001) are retained byte-for-byte as legacy records. Their extra fields and old status vocabulary are not evidence of an ACK. All newly appended events use the eight-field envelope above. Send a fresh handshake under this protocol; awaiting status refers to that handshake. Legacy job evidence remains unverified and is not merged into the SSOT by this initialization.

ChatGPT uses a dedicated repository-scoped SSH deploy key for direct Git pull/push; browser interaction and a Gist ACK channel are not required. The GitHub connector previously returned HTTP 403 on writes and is not the write transport. Credentials remain outside this repository. ACK processing still requires an active agent run; this setup does not install a scheduler. After ChatGPT sends the handshake ACK, Doubao is AWAITING_RECEIPT until its receipt is read from remote main.
