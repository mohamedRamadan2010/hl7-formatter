# HL7 Formatter

A single-file, browser-based tool for cleaning up and re-identifying HL7 v2 messages.
Paste a raw message, click **Format**, and get readable, segment-per-line output that you
can copy straight into the HAPI TestPanel — one message at a time or as a whole batch.

It runs entirely in the browser. Nothing is uploaded or sent anywhere; open the HTML file
locally and it works offline.

## What it does

- **Splits a message into segments**, one per line. Works whether the input is all on one
  line with `\r` separators, already partly line-broken, or has segments joined by a stray
  space (e.g. `...NO KNOWN ALLERGIES DG1|||...`) — a case a plain `\r`-based split misses.
- **Separates batches by message.** Each `MSH` starts a new message block, shown with a
  divider and a *Message N* label.
- **Colour-codes segments by category** (control, patient, visit, orders/observations,
  allergy/diagnosis, insurance) and dims the `|` separators so the structure is easy to scan.
- **Rewrites identifiers across all messages** — MRN, Visit/CSN, Message ID, Order ID — with
  a value you type or one generated for you. Useful for re-sending captured messages without
  tripping duplicate detection.
- **Copies per message or per batch**, `\r`-terminated and ready for HAPI.

## Getting started

1. Open `hl7-formatter.html` in any modern browser (Chrome, Edge, Firefox, Safari 16.4+).
2. Paste an HL7 message into the left pane, or click **Load sample**.
3. Click **Format** (or press `Ctrl/Cmd + Enter`).
4. Read the formatted result on the right, and use the copy buttons.

## The interface

### Input pane
- A text area for the raw message.
- **Load sample** drops in an example ADT^A08 message (including a space-joined `AL1/DG1`
  run-on) so you can see the splitting at work.
- **Clear** empties the input.

### Output pane
- Segments grouped per message, colour-coded, with separators dimmed.
- Each message has its own **Copy** button that copies just that message,
  `\r`-terminated for HAPI.
- **Copy** (top right) copies the whole batch in readable form (`\n`, with a blank line
  between messages).
- **Copy for HAPI** copies the whole batch `\r`-terminated, without the blank boundary lines
  (a blank line is an empty segment that HAPI's parser rejects).

### Status bar
Shows the message count, the segment count, and whether the first segment is `MSH`.

### Extra segments
A comma-separated list of additional segment IDs (e.g. site-specific Z-segments like
`ZBE, ZPD`) so they are recognised as segment boundaries and given their own line.

## Overrides

Each row rewrites an identifier in every message when you click **Format**.

| Field         | Default target  | Per message | Notes |
|---------------|-----------------|-------------|-------|
| MRN           | `PID-3`         | off         | Same value across all messages. |
| Visit / CSN   | `PV1-19,PID-18` | off         | Updates both the visit number and patient account number. |
| Message ID    | `MSH-10`        | on          | Replaces only the number, keeps surrounding text (see below). |
| Order ID      | `ORC-2,OBR-2`   | on          | Consistent within a message, unique across messages. |

Each row has:

- **Enable** checkbox — apply this override or not.
- **Target field(s)** — an editable `SEG-N` reference, or several comma-separated. Adjust if
  your interface carries an ID in a different field (for example add `PID-18` for an account
  number, or change a Z-segment field).
- **Value** — the replacement value you type.
- **Auto** — generates a value, enables the row, and reformats. Handy for fresh IDs on resend.
- **per msg** — when on, appends `-1`, `-2`, … per message so each message gets a unique value.
  Leave it off to apply the same value everywhere (e.g. one MRN for the whole patient).

### Replacement behaviour
- Only the **first component** of a field is replaced, so an MRN like `378595^^^MR` keeps its
  `^^^MR` suffix.
- **Message ID keeps its text.** If `MSH-10` is `106796ANC`, overriding the number gives
  `555000ANC` — only the numeric run changes, the `ANC` tag is preserved. With *per msg* on,
  you get `555000-1ANC`, `555000-2ANC`, … (unique and still tagged). A purely numeric ID is
  replaced in full.

## HL7 field numbering

Targets use standard HL7 field numbers (`PID-3` = the third field of a `PID` segment).
`MSH` is offset by one because `MSH-1` is the field separator itself, so `MSH-10` is the
message control ID. The tool handles this offset for you; just use the standard field number.

## Notes and limits

- This is **structural formatting and field replacement only**, not schema validation. Real
  validation is HAPI's job — format here, then parse with `PipeParser`.
- Overrides write by **field position**, which assumes well-formed segments. This is correct
  for standard Epic messages, but a structurally broken `MSH` could shift positions, so format
  first and check the output before trusting a bulk override.
- The HAPI TestPanel parses one message at a time; for a batch, use the per-message **Copy**.
- All processing is local to the browser. No network calls, no storage.

## Related

The same normalisation logic exists as a dependency-free Java class (`Hl7Formatter`) for use
in backend tooling, with an optional HAPI-based validator. Keeping the two in sync means the
browser tool and the server produce identical output.
