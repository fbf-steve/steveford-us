# Client Intake Form — Design Spec

Date: 2026-07-17

## Purpose

A standalone form on steveford.us that a prospective client fills out to describe
their business. The submitted answers give Steve everything needed to draft a
receptionist-agent system prompt for them (the same kind of prompt used to build
the "Ava" agent for the Summit Comfort demo). The form does not generate the
prompt itself — it collects structured answers and emails them to Steve, who
turns them into a prompt in a follow-up session.

## Architecture

- New file `intake.html` at the repo root, alongside the existing `index.html`.
  Deploys with the same Cloudflare Pages setup — no config changes.
- Self-contained: inline `<style>` and `<script>`, no external JS dependencies
  except the Web3Forms submit endpoint. Reuses `index.html`'s existing CSS
  variables (`--navy`, `--accent`, `--accent-grad`, etc.) so it matches the
  site's look.
- State: a single JS object `formData` holds all answers keyed by field name,
  updated on input. A `currentStep` integer (0–6: 6 content steps + review)
  controls which `<section class="step">` is visible, reflected in a progress
  bar.
- Navigation: Next/Back buttons per step. "Next" validates only that step's
  required fields before advancing. Back always allowed, preserves entered
  values.
- Final step: read-only **Review & Submit** screen listing everything entered,
  grouped by step, each group with an "Edit" link that jumps back to that step.

## Fields per step

**Step 1 — Business Basics**
`business_name`, `industry` (short text, e.g. "HVAC & plumbing"), `contact_name`,
`contact_email`, `contact_phone`, `website` (optional)

**Step 2 — Services & Pricing**
`services_offered` (textarea), `pricing_notes` (textarea — service fees, free
estimates, financing, what the agent may/may not quote), `standard_service_fee`
(optional short text, e.g. "$89, waived if repair proceeds")

**Step 3 — Hours & Service Area**
`business_hours` (textarea, e.g. "Mon–Fri 7:30am–5:30pm"), `timezone`,
`service_area` (textarea — cities/radius/zip codes), `after_hours_policy`
(textarea)

**Step 4 — Call Handling**
`agent_capabilities` (checkboxes: Book appointments / Take messages / Answer
FAQs / Transfer calls), `booking_info_needed` (checkboxes, pre-filled with
defaults — name, callback number, address, problem description, timing —
editable), `transfer_destinations` (repeatable rows: department name + phone
number, add/remove row, can be empty)

**Step 5 — Emergencies & Escalation**
`has_emergencies` (yes/no), `emergency_definition` (textarea), `emergency_script`
(textarea, optional — required safety language, e.g. gas-leak instructions),
`oncall_process` (textarea)

**Step 6 — Tone & Voice**
`agent_name_preference` (short text or "no preference"), `tone` (multiple
choice: Warm & casual / Professional & polished / Playful / Formal),
`voice_gender_preference` (Male / Female / No preference), `never_say`
(textarea — hard boundaries)

**Review & Submit**: read-only summary of all of the above, grouped by step,
with Edit links, then Submit.

Required fields (block "Next" until filled): `business_name`, `contact_name`,
`contact_email`, `contact_phone`. All other fields are optional/non-blocking.

## Submission & error handling

- On Review screen submit, `fetch(POST)` to `https://api.web3forms.com/submit`
  with `access_key` + all `formData` fields serialized as JSON
  (`Content-Type: application/json`). No page reload.
- Spam protection: a hidden honeypot field using Web3Forms' `botcheck`
  convention — invisible to real users, submissions with it filled are
  silently discarded by Web3Forms.
- Success: replace the form with an inline confirmation message ("Thanks —
  I'll follow up within a business day."). No separate thank-you page.
- Failure (network error or non-2xx response): show an inline error with a
  "Copy my answers" button (copies a plain-text summary of all entered
  answers to the clipboard) plus a `mailto:` link pre-addressed to Steve, so a
  submission is never silently lost.
- Client-side validation: per-step required-field checks only; no whole-form
  up-front validation.

## Testing plan

No build step means no automated test framework — this gets a manual QA pass
before considering the form done:

- Each step's "Next" is blocked until that step's required fields are filled
- Back/Next preserves previously entered values
- Repeatable transfer-destination rows can be added and removed cleanly,
  including down to zero rows
- Review screen accurately reflects every field entered; each "Edit" link
  jumps to the correct step
- A real test submission arrives at Steve's email via Web3Forms
- Simulated failure (e.g. wrong access key) triggers the fallback
  (copy-to-clipboard + mailto) instead of failing silently
- Mobile viewport (narrow width) — steps, buttons, and progress bar remain
  usable
- Browser console shows no JS errors through a full run-through

## Out of scope

- Auto-generating the agent prompt from submitted answers (a possible future
  iteration, not this build)
- Any backend/database storage of submissions beyond what Web3Forms itself
  retains
- Authentication/login — this is a public lead-gen form
