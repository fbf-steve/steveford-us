# Client Intake Form Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `intake.html`, a standalone 6-step wizard form on steveford.us that prospective clients fill out to describe their business, so Steve has everything needed to draft a receptionist-agent prompt for them.

**Architecture:** A single self-contained static HTML file (inline CSS + JS, no build step, no dependencies) added to the existing `steveford-us` Cloudflare Pages repo. A small vanilla-JS wizard engine drives step navigation and a generic `formData` object; a data-driven `FIELD_GROUPS` config powers both step rendering hooks and the review screen. Submission goes to Web3Forms via `fetch`.

**Tech Stack:** HTML5, vanilla JavaScript (no framework), CSS (reusing `index.html`'s existing custom-property design tokens), Web3Forms (https://web3forms.com) as the email-delivery backend.

## Global Constraints

- No build step — `intake.html` must be a single file with inline `<style>`/`<script>`, deployable as-is on Cloudflare Pages (per spec Architecture).
- Reuse `index.html`'s existing CSS custom properties (`--navy`, `--navy-2`, `--ink`, `--paper`, `--accent`, `--accent-2`, `--accent-grad`, `--muted`, `--line`, `--card`, `--shadow`, `--radius`) so the form matches the site (per spec Architecture).
- Only `business_name`, `contact_name`, `contact_email`, `contact_phone` (all in Step 1) are required; every other field is optional and never blocks "Next" (per spec Fields section).
- Submission must go to `https://api.web3forms.com/submit` via `fetch(POST)` with `Content-Type: application/json`, no page reload (per spec Submission & error handling).
- Must include a Web3Forms honeypot field (`botcheck` convention) for spam protection (per spec Submission & error handling).
- On submit failure, must show a "Copy my answers" fallback (clipboard) plus a `mailto:` link — a submission must never be silently lost (per spec Submission & error handling).
- Out of scope: auto-generating the agent prompt, any backend/database beyond Web3Forms, authentication (per spec Out of scope).

---

### Task 1: Page shell, wizard engine, Business Basics step, and Review screen scaffold

**Files:**
- Create: `intake.html`

**Interfaces:**
- Produces: `formData` (global object, keyed by field `name`), `FIELD_GROUPS` (array of `{step: number, title: string, fields: [{key: string, label: string}]}>`), `REQUIRED_FIELDS` (object mapping step index → array of required field keys), `showStep(n)`, `nextStep()`, `prevStep()`, `goToStep(n)`, `validateStep(n)`, `bindInputs()`, `updateFormData(el)`, `renderReview()`, `escapeHtml(str)` — all later tasks call `FIELD_GROUPS.push(...)` to register their step's fields and rely on `bindInputs()` having already auto-wired any `[name]` element present in the DOM at load time.

- [ ] **Step 1: Write `intake.html` with page shell, CSS, and the wizard engine**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Tell Us About Your Business — Steve Ford</title>
<meta name="description" content="Tell us about your business so we can draft your AI receptionist agent.">
<link rel="canonical" href="https://steveford.us/intake.html">
<style>
  :root{
    --navy:#0a1f3c;
    --navy-2:#0d2b52;
    --ink:#0b1220;
    --paper:#f7f9fc;
    --accent:#2f81f7;
    --accent-2:#38d9c4;
    --accent-grad:linear-gradient(135deg,#2f81f7 0%,#38d9c4 100%);
    --muted:#5b6b82;
    --line:rgba(255,255,255,.10);
    --card:#ffffff;
    --shadow:0 10px 40px rgba(10,31,60,.10);
    --radius:16px;
  }
  *{box-sizing:border-box;margin:0;padding:0}
  body{
    font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Inter,Roboto,Helvetica,Arial,sans-serif;
    color:var(--ink);background:var(--paper);line-height:1.6;
    -webkit-font-smoothing:antialiased;
  }
  a{color:inherit}
  .wrap{max-width:720px;margin:0 auto;padding:48px 24px 96px}
  header.top{background:linear-gradient(180deg,var(--navy) 0%,var(--navy-2) 100%);color:#fff;padding:40px 0}
  header.top .wrap{padding-top:0;padding-bottom:0}
  header.top h1{font-size:clamp(1.6rem,4vw,2.2rem);letter-spacing:-.02em;font-weight:800}
  header.top p{color:#cdd9ec;margin-top:10px;max-width:56ch}

  .progress{display:flex;gap:8px;margin:32px 0}
  .progress-dot{flex:1;height:6px;border-radius:999px;background:#e2e8f2}
  .progress-dot.done{background:var(--accent-grad)}
  .progress-dot.current{background:var(--accent)}

  .card{background:var(--card);border:1px solid #e8edf5;border-radius:var(--radius);padding:32px;box-shadow:var(--shadow)}
  .step{display:none}
  .step.active{display:block}
  .step h2{font-size:1.4rem;letter-spacing:-.01em;margin-bottom:6px}
  .step .step-sub{color:var(--muted);margin-bottom:24px}

  label{display:block;font-weight:600;font-size:.92rem;margin:18px 0 6px}
  label:first-of-type{margin-top:0}
  input[type=text],input[type=email],input[type=tel],input[type=url],textarea,select{
    width:100%;border:1px solid #d7e0ec;border-radius:10px;padding:11px 13px;font-size:.98rem;font-family:inherit;color:var(--ink);
  }
  textarea{min-height:90px;resize:vertical}
  input:focus,textarea:focus,select:focus{outline:2px solid var(--accent);outline-offset:1px}
  .field-hint{color:var(--muted);font-size:.85rem;margin-top:4px}
  .checkbox-row,.radio-row{display:flex;align-items:center;gap:8px;margin:8px 0}
  .checkbox-row label,.radio-row label{margin:0;font-weight:500}

  .btn{display:inline-flex;align-items:center;gap:8px;font-weight:700;padding:12px 24px;border-radius:999px;font-size:.96rem;border:none;cursor:pointer;font-family:inherit}
  .btn-primary{background:var(--accent-grad);color:#04121f}
  .btn-ghost{background:transparent;border:1px solid #d7e0ec;color:var(--ink)}
  .nav-row{display:flex;justify-content:space-between;margin-top:32px}
  .nav-row .spacer{flex:1}

  .review-group{border-top:1px solid #e8edf5;padding:16px 0}
  .review-group:first-child{border-top:none;padding-top:0}
  .review-group-head{display:flex;justify-content:space-between;align-items:center}
  .review-group-head h3{font-size:1rem}
  .review-edit{color:var(--accent);font-weight:600;font-size:.88rem}
  .review-row{display:flex;gap:10px;padding:6px 0;font-size:.92rem}
  .review-label{color:var(--muted);min-width:160px;flex:0 0 auto}
  .review-value{white-space:pre-wrap}
  .review-empty{color:var(--muted);font-size:.88rem;padding:6px 0}

  .confirm{text-align:center;padding:40px 0}
  .confirm h2{font-size:1.5rem}
  .confirm p{color:var(--muted);margin-top:10px}
  .error-box{background:#fff4f2;border:1px solid #f3c9c2;border-radius:12px;padding:18px;margin-top:20px}
  .error-box p{margin-bottom:12px}
</style>
</head>
<body>

<header class="top">
  <div class="wrap">
    <h1>Tell us about your business</h1>
    <p>A few quick questions so we can draft the AI receptionist agent for your business — like the one we built for Summit Comfort.</p>
  </div>
</header>

<div class="wrap">
  <div class="progress" id="progress">
    <div class="progress-dot" data-step="0"></div>
    <div class="progress-dot" data-step="1"></div>
    <div class="progress-dot" data-step="2"></div>
    <div class="progress-dot" data-step="3"></div>
    <div class="progress-dot" data-step="4"></div>
    <div class="progress-dot" data-step="5"></div>
    <div class="progress-dot" data-step="6"></div>
  </div>

  <div class="card">
    <form id="intake-form">

      <section class="step" data-step="0">
        <h2>Business Basics</h2>
        <p class="step-sub">Who are you and how do we reach you?</p>

        <label for="business_name">Business name *</label>
        <input type="text" name="business_name" id="business_name">

        <label for="industry">What does your business do? (short description)</label>
        <input type="text" name="industry" id="industry" placeholder="e.g. HVAC &amp; plumbing">

        <label for="contact_name">Your name *</label>
        <input type="text" name="contact_name" id="contact_name">

        <label for="contact_email">Your email *</label>
        <input type="email" name="contact_email" id="contact_email">

        <label for="contact_phone">Your phone *</label>
        <input type="tel" name="contact_phone" id="contact_phone">

        <label for="website">Website (optional)</label>
        <input type="url" name="website" id="website" placeholder="https://">

        <div class="nav-row"><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>

      <section class="step" data-step="1">
        <h2>Services &amp; Pricing</h2>
        <p class="step-sub">Added in a later task.</p>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>

      <section class="step" data-step="2">
        <h2>Hours &amp; Service Area</h2>
        <p class="step-sub">Added in a later task.</p>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>

      <section class="step" data-step="3">
        <h2>Call Handling</h2>
        <p class="step-sub">Added in a later task.</p>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>

      <section class="step" data-step="4">
        <h2>Emergencies &amp; Escalation</h2>
        <p class="step-sub">Added in a later task.</p>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>

      <section class="step" data-step="5">
        <h2>Tone &amp; Voice</h2>
        <p class="step-sub">Added in a later task.</p>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>

      <section class="step" data-step="6">
        <h2>Review &amp; Submit</h2>
        <p class="step-sub">Check everything looks right, then send it over.</p>
        <div id="review-content"></div>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" id="submit-btn">Submit</button></div>
      </section>

    </form>
  </div>
</div>

<script>
  const formData = {};
  let currentStep = 0;

  const FIELD_GROUPS = [
    { step: 0, title: 'Business Basics', fields: [
      { key: 'business_name', label: 'Business name' },
      { key: 'industry', label: 'What they do' },
      { key: 'contact_name', label: 'Contact name' },
      { key: 'contact_email', label: 'Contact email' },
      { key: 'contact_phone', label: 'Contact phone' },
      { key: 'website', label: 'Website' },
    ]},
  ];

  const REQUIRED_FIELDS = {
    0: ['business_name', 'contact_name', 'contact_email', 'contact_phone'],
  };

  function escapeHtml(str) {
    const div = document.createElement('div');
    div.textContent = str == null ? '' : String(str);
    return div.innerHTML;
  }

  function updateFormData(el) {
    const name = el.name;
    if (!name) return;
    if (el.type === 'checkbox') {
      if (!Array.isArray(formData[name])) formData[name] = [];
      if (el.checked) {
        if (!formData[name].includes(el.value)) formData[name].push(el.value);
      } else {
        formData[name] = formData[name].filter(v => v !== el.value);
      }
    } else if (el.type === 'radio') {
      if (el.checked) formData[name] = el.value;
    } else {
      formData[name] = el.value;
    }
  }

  function bindInputs() {
    document.querySelectorAll('[name]').forEach(el => {
      const evt = (el.type === 'checkbox' || el.type === 'radio') ? 'change' : 'input';
      el.addEventListener(evt, () => updateFormData(el));
      updateFormData(el); // seed formData with any pre-set defaults
    });
  }

  function validateStep(n) {
    const required = REQUIRED_FIELDS[n] || [];
    return required.every(key => (formData[key] || '').trim().length > 0);
  }

  function showStep(n) {
    document.querySelectorAll('.step').forEach(sec => {
      sec.classList.toggle('active', Number(sec.dataset.step) === n);
    });
    document.querySelectorAll('.progress-dot').forEach(dot => {
      const s = Number(dot.dataset.step);
      dot.classList.toggle('done', s < n);
      dot.classList.toggle('current', s === n);
    });
    currentStep = n;
    if (n === 6) renderReview();
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }

  function nextStep() {
    if (!validateStep(currentStep)) {
      alert('Please fill in all required fields before continuing.');
      return;
    }
    showStep(Math.min(currentStep + 1, 6));
  }

  function prevStep() {
    showStep(Math.max(currentStep - 1, 0));
  }

  function goToStep(n) {
    showStep(n);
  }

  function renderReview() {
    const container = document.getElementById('review-content');
    container.innerHTML = FIELD_GROUPS.map(group => {
      const rows = group.fields.map(f => {
        let value = formData[f.key];
        if (Array.isArray(value)) value = value.join(', ');
        if (!value) return '';
        return `<div class="review-row"><span class="review-label">${escapeHtml(f.label)}</span><span class="review-value">${escapeHtml(value)}</span></div>`;
      }).join('');
      return `<div class="review-group">
        <div class="review-group-head"><h3>${escapeHtml(group.title)}</h3><a href="#" class="review-edit" data-step="${group.step}">Edit</a></div>
        ${rows || '<p class="review-empty">No answers yet.</p>'}
      </div>`;
    }).join('');
  }

  document.addEventListener('click', (e) => {
    if (e.target.matches('.review-edit')) {
      e.preventDefault();
      goToStep(Number(e.target.dataset.step));
    }
  });

  bindInputs();
  showStep(0);
</script>
</body>
</html>
```

- [ ] **Step 2: Manually verify Step 1 and Review scaffold work**

Open `intake.html` directly in a browser (double-click the file, or `open intake.html` on macOS).

Verify, in order:
1. "Business Basics" step is visible; progress bar shows 7 dots, first one styled `current`.
2. Click "Next" without filling anything → an alert appears ("Please fill in all required fields...") and the step does not advance.
3. Fill in Business name, Your name, Your email, Your phone (leave Industry/Website blank). Click "Next" → advances to the "Services & Pricing" placeholder step ("Added in a later task."), progress dot 0 now shows `done`.
4. Click "Back" → returns to Business Basics with all four values still filled in.
5. Click "Next" through the remaining placeholder steps (Services & Pricing → Hours & Service Area → Call Handling → Emergencies & Escalation → Tone & Voice) to reach "Review & Submit".
6. On Review, confirm a "Business Basics" group appears showing the four filled values with correct labels, and "Edit" is a visible link.
7. Click "Edit" next to Business Basics → returns to step 0 with values intact.

Expected: all of the above match exactly. If the alert doesn't block, or values are lost on Back, fix `validateStep`/`updateFormData` before proceeding.

- [ ] **Step 3: Commit**

```bash
cd ~/development/steveford-us
git add intake.html
git commit -m "Add intake form scaffold: wizard engine, Business Basics step, review screen"
```

---

### Task 2: Services & Pricing step

**Files:**
- Modify: `intake.html` — replace the `data-step="1"` section's placeholder content, extend `FIELD_GROUPS`

**Interfaces:**
- Consumes: `FIELD_GROUPS` (array, push a new group), `bindInputs()` (already runs once at page load in Task 1 — do not call it again; the fields added here are wired automatically since `bindInputs` will have already run against the full DOM by the time the browser finishes loading the `<script>` block, as long as this HTML is added before the closing `</script>`'s `bindInputs()` call in document order — it is, since it's part of the same static markup).
- Produces: `formData.services_offered`, `formData.pricing_notes`, `formData.standard_service_fee`.

- [ ] **Step 1: Replace the Services & Pricing section's placeholder content**

Find this block in `intake.html`:

```html
      <section class="step" data-step="1">
        <h2>Services &amp; Pricing</h2>
        <p class="step-sub">Added in a later task.</p>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>
```

Replace it with:

```html
      <section class="step" data-step="1">
        <h2>Services &amp; Pricing</h2>
        <p class="step-sub">What do you do, and how should the agent talk about money?</p>

        <label for="services_offered">Services you offer</label>
        <textarea name="services_offered" id="services_offered" placeholder="e.g. AC repair, furnace repair, water heater installs, drain cleaning..."></textarea>

        <label for="pricing_notes">Pricing notes for the agent</label>
        <textarea name="pricing_notes" id="pricing_notes" placeholder="e.g. Diagnostic fee is $89, waived if repair proceeds. Free estimates on new installs. Financing available. Never quote exact repair prices."></textarea>
        <p class="field-hint">This becomes the "what the agent can and can't say about money" section of the prompt.</p>

        <label for="standard_service_fee">Standard service/diagnostic fee (optional)</label>
        <input type="text" name="standard_service_fee" id="standard_service_fee" placeholder="e.g. $89, waived if repair proceeds">

        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>
```

- [ ] **Step 2: Extend `FIELD_GROUPS`**

Find:

```js
  const FIELD_GROUPS = [
    { step: 0, title: 'Business Basics', fields: [
      { key: 'business_name', label: 'Business name' },
      { key: 'industry', label: 'What they do' },
      { key: 'contact_name', label: 'Contact name' },
      { key: 'contact_email', label: 'Contact email' },
      { key: 'contact_phone', label: 'Contact phone' },
      { key: 'website', label: 'Website' },
    ]},
  ];
```

Replace with:

```js
  const FIELD_GROUPS = [
    { step: 0, title: 'Business Basics', fields: [
      { key: 'business_name', label: 'Business name' },
      { key: 'industry', label: 'What they do' },
      { key: 'contact_name', label: 'Contact name' },
      { key: 'contact_email', label: 'Contact email' },
      { key: 'contact_phone', label: 'Contact phone' },
      { key: 'website', label: 'Website' },
    ]},
    { step: 1, title: 'Services & Pricing', fields: [
      { key: 'services_offered', label: 'Services offered' },
      { key: 'pricing_notes', label: 'Pricing notes' },
      { key: 'standard_service_fee', label: 'Standard service fee' },
    ]},
  ];
```

- [ ] **Step 3: Manually verify**

Reload `intake.html` in the browser. Fill Step 1 required fields, click Next. On "Services & Pricing", fill `services_offered` and `pricing_notes` (leave `standard_service_fee` blank). Click Next through to Review.

Expected: a "Services & Pricing" group appears on Review with the two filled values shown and `standard_service_fee` simply absent (not shown as empty/blank row). Clicking its "Edit" link returns to step 1 with values intact.

- [ ] **Step 4: Commit**

```bash
cd ~/development/steveford-us
git add intake.html
git commit -m "Add Services & Pricing step to intake form"
```

---

### Task 3: Hours & Service Area step

**Files:**
- Modify: `intake.html` — replace the `data-step="2"` section's placeholder content, extend `FIELD_GROUPS`

**Interfaces:**
- Consumes: same `FIELD_GROUPS` push pattern as Task 2.
- Produces: `formData.business_hours`, `formData.timezone`, `formData.service_area`, `formData.after_hours_policy`.

- [ ] **Step 1: Replace the Hours & Service Area section's placeholder content**

Find:

```html
      <section class="step" data-step="2">
        <h2>Hours &amp; Service Area</h2>
        <p class="step-sub">Added in a later task.</p>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>
```

Replace with:

```html
      <section class="step" data-step="2">
        <h2>Hours &amp; Service Area</h2>
        <p class="step-sub">When are you open, and where do you work?</p>

        <label for="business_hours">Business hours</label>
        <textarea name="business_hours" id="business_hours" placeholder="e.g. Monday-Friday, 7:30am-5:30pm"></textarea>

        <label for="timezone">Timezone</label>
        <input type="text" name="timezone" id="timezone" placeholder="e.g. Central">

        <label for="service_area">Service area</label>
        <textarea name="service_area" id="service_area" placeholder="e.g. 30-mile radius around Lebanon, TN - Wilson County, Mt. Juliet, Hermitage, Nashville"></textarea>

        <label for="after_hours_policy">After-hours / weekend policy</label>
        <textarea name="after_hours_policy" id="after_hours_policy" placeholder="e.g. Weekends are emergency-only. 24/7 emergency line available."></textarea>

        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>
```

- [ ] **Step 2: Extend `FIELD_GROUPS`**

Append after the `Services & Pricing` group entry added in Task 2 (inside the same `FIELD_GROUPS` array literal):

```js
    { step: 2, title: 'Hours & Service Area', fields: [
      { key: 'business_hours', label: 'Business hours' },
      { key: 'timezone', label: 'Timezone' },
      { key: 'service_area', label: 'Service area' },
      { key: 'after_hours_policy', label: 'After-hours policy' },
    ]},
```

- [ ] **Step 3: Manually verify**

Reload, fill Step 1 required fields, click through to "Hours & Service Area", fill all four fields, click through to Review.

Expected: "Hours & Service Area" group appears on Review with all four values; Edit link returns to step 2 with values intact.

- [ ] **Step 4: Commit**

```bash
cd ~/development/steveford-us
git add intake.html
git commit -m "Add Hours & Service Area step to intake form"
```

---

### Task 4: Call Handling step (capabilities, booking info, repeatable transfer numbers)

**Files:**
- Modify: `intake.html` — replace the `data-step="3"` section's placeholder content, extend `FIELD_GROUPS`, add repeatable-row JS

**Interfaces:**
- Consumes: `escapeHtml(str)` (from Task 1), same `FIELD_GROUPS` push pattern.
- Produces: `formData.agent_capabilities` (array), `formData.booking_info_needed` (array), `formData.transfer_destinations` (array of `"Dept: number"` strings), `addTransferRow(dept, number)`, `removeTransferRow(id)`, `renderTransferRows()`, `syncTransferData()`.

This step needs its own logic because `transfer_destinations` is a repeatable list (add/remove rows), which the generic single-value `[name]` binder from Task 1 doesn't handle — each row's inputs aren't present in the DOM until the user clicks "Add".

- [ ] **Step 1: Replace the Call Handling section's placeholder content**

Find:

```html
      <section class="step" data-step="3">
        <h2>Call Handling</h2>
        <p class="step-sub">Added in a later task.</p>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>
```

Replace with:

```html
      <section class="step" data-step="3">
        <h2>Call Handling</h2>
        <p class="step-sub">What should the agent be able to do on a call?</p>

        <label>The agent should be able to:</label>
        <div class="checkbox-row"><input type="checkbox" name="agent_capabilities" value="Book appointments" id="cap_book"><label for="cap_book">Book appointments</label></div>
        <div class="checkbox-row"><input type="checkbox" name="agent_capabilities" value="Take messages" id="cap_message"><label for="cap_message">Take messages</label></div>
        <div class="checkbox-row"><input type="checkbox" name="agent_capabilities" value="Answer FAQs" id="cap_faq"><label for="cap_faq">Answer FAQs</label></div>
        <div class="checkbox-row"><input type="checkbox" name="agent_capabilities" value="Transfer calls" id="cap_transfer"><label for="cap_transfer">Transfer calls</label></div>

        <label>When booking, collect:</label>
        <div class="checkbox-row"><input type="checkbox" name="booking_info_needed" value="Name" id="book_name" checked><label for="book_name">Name</label></div>
        <div class="checkbox-row"><input type="checkbox" name="booking_info_needed" value="Callback number" id="book_number" checked><label for="book_number">Callback number</label></div>
        <div class="checkbox-row"><input type="checkbox" name="booking_info_needed" value="Address" id="book_address" checked><label for="book_address">Address</label></div>
        <div class="checkbox-row"><input type="checkbox" name="booking_info_needed" value="Problem description" id="book_problem" checked><label for="book_problem">Problem description</label></div>
        <div class="checkbox-row"><input type="checkbox" name="booking_info_needed" value="Preferred timing" id="book_timing" checked><label for="book_timing">Preferred timing</label></div>

        <label>Transfer numbers (optional)</label>
        <p class="field-hint">Add a row for each department/person calls might get transferred to.</p>
        <div id="transfer-rows"></div>
        <button type="button" class="btn btn-ghost" id="add-transfer-row" style="margin-top:10px">+ Add transfer number</button>

        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>
```

- [ ] **Step 2: Extend `FIELD_GROUPS`**

Append after the `Hours & Service Area` group entry added in Task 3:

```js
    { step: 3, title: 'Call Handling', fields: [
      { key: 'agent_capabilities', label: 'Agent can' },
      { key: 'booking_info_needed', label: 'Collects when booking' },
      { key: 'transfer_destinations', label: 'Transfer numbers' },
    ]},
```

- [ ] **Step 3: Add repeatable transfer-row JS**

Find this line in `intake.html` (the end of the script's setup, right before the final `bindInputs(); showStep(0);` calls):

```js
  document.addEventListener('click', (e) => {
    if (e.target.matches('.review-edit')) {
      e.preventDefault();
      goToStep(Number(e.target.dataset.step));
    }
  });

  bindInputs();
  showStep(0);
```

Replace with:

```js
  document.addEventListener('click', (e) => {
    if (e.target.matches('.review-edit')) {
      e.preventDefault();
      goToStep(Number(e.target.dataset.step));
    }
  });

  let transferRows = [];

  function addTransferRow(dept = '', number = '') {
    transferRows.push({ id: transferRows.length ? Math.max(...transferRows.map(r => r.id)) + 1 : 1, dept, number });
    renderTransferRows();
  }

  function removeTransferRow(id) {
    transferRows = transferRows.filter(r => r.id !== id);
    renderTransferRows();
  }

  function syncTransferData() {
    formData.transfer_destinations = transferRows
      .filter(r => r.dept || r.number)
      .map(r => `${r.dept}: ${r.number}`);
  }

  function renderTransferRows() {
    const container = document.getElementById('transfer-rows');
    container.innerHTML = transferRows.map(r => `
      <div class="checkbox-row" data-id="${r.id}" style="gap:10px">
        <input type="text" placeholder="Department (e.g. Sales)" class="transfer-dept" value="${escapeHtml(r.dept)}" style="width:45%">
        <input type="tel" placeholder="Phone number" class="transfer-number" value="${escapeHtml(r.number)}" style="width:45%">
        <button type="button" class="btn btn-ghost transfer-remove" data-id="${r.id}">Remove</button>
      </div>
    `).join('');
    container.querySelectorAll('.transfer-dept').forEach(el => {
      const id = Number(el.closest('[data-id]').dataset.id);
      el.addEventListener('input', () => {
        transferRows.find(r => r.id === id).dept = el.value;
        syncTransferData();
      });
    });
    container.querySelectorAll('.transfer-number').forEach(el => {
      const id = Number(el.closest('[data-id]').dataset.id);
      el.addEventListener('input', () => {
        transferRows.find(r => r.id === id).number = el.value;
        syncTransferData();
      });
    });
    container.querySelectorAll('.transfer-remove').forEach(el => {
      el.addEventListener('click', () => removeTransferRow(Number(el.dataset.id)));
    });
    syncTransferData();
  }

  document.getElementById('add-transfer-row').addEventListener('click', () => addTransferRow());

  bindInputs();
  showStep(0);
```

- [ ] **Step 4: Manually verify**

Reload, click through Steps 1-2 filling required fields only, reach "Call Handling". Check "Book appointments" and "Transfer calls". Confirm the five "collect when booking" boxes are checked by default. Click "+ Add transfer number" twice; fill row 1 as `Dispatch` / `615-555-0101`, row 2 as `Sales` / `615-555-0102`. Click "Remove" on row 2. Click Next through to Review.

Expected: Review's "Call Handling" group shows `Agent can: Book appointments, Transfer calls`, `Collects when booking: Name, Callback number, Address, Problem description, Preferred timing`, and `Transfer numbers: Dispatch: 615-555-0101` (row 2 excluded since it was removed). Edit link returns to step 3 with the one remaining transfer row and all checkboxes intact.

- [ ] **Step 5: Commit**

```bash
cd ~/development/steveford-us
git add intake.html
git commit -m "Add Call Handling step with repeatable transfer numbers to intake form"
```

---

### Task 5: Emergencies & Escalation step

**Files:**
- Modify: `intake.html` — replace the `data-step="4"` section's placeholder content, extend `FIELD_GROUPS`

**Interfaces:**
- Consumes: same `FIELD_GROUPS` push pattern.
- Produces: `formData.has_emergencies`, `formData.emergency_definition`, `formData.emergency_script`, `formData.oncall_process`.

- [ ] **Step 1: Replace the Emergencies & Escalation section's placeholder content**

Find:

```html
      <section class="step" data-step="4">
        <h2>Emergencies &amp; Escalation</h2>
        <p class="step-sub">Added in a later task.</p>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>
```

Replace with:

```html
      <section class="step" data-step="4">
        <h2>Emergencies &amp; Escalation</h2>
        <p class="step-sub">Does your business have true emergencies, and how should the agent handle them?</p>

        <label>Does your business have true emergencies?</label>
        <div class="radio-row"><input type="radio" name="has_emergencies" value="Yes" id="emerg_yes"><label for="emerg_yes">Yes</label></div>
        <div class="radio-row"><input type="radio" name="has_emergencies" value="No" id="emerg_no"><label for="emerg_no">No</label></div>

        <label for="emergency_definition">What counts as an emergency for your business?</label>
        <textarea name="emergency_definition" id="emergency_definition" placeholder="e.g. No heat/cooling in extreme weather, burst pipe, gas smell, sewage backup"></textarea>

        <label for="emergency_script">Required safety language (optional)</label>
        <textarea name="emergency_script" id="emergency_script" placeholder="e.g. If a caller reports a gas smell: tell them to leave the house immediately, not touch any switches, and call 911 and the gas company from outside."></textarea>

        <label for="oncall_process">On-call / escalation process</label>
        <textarea name="oncall_process" id="oncall_process" placeholder="e.g. Emergency is flagged and the on-call technician is notified immediately."></textarea>

        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>
```

- [ ] **Step 2: Extend `FIELD_GROUPS`**

Append after the `Call Handling` group entry added in Task 4:

```js
    { step: 4, title: 'Emergencies & Escalation', fields: [
      { key: 'has_emergencies', label: 'Has true emergencies' },
      { key: 'emergency_definition', label: 'What counts as an emergency' },
      { key: 'emergency_script', label: 'Required safety language' },
      { key: 'oncall_process', label: 'On-call process' },
    ]},
```

- [ ] **Step 3: Manually verify**

Reload, fill Step 1 required fields, click through to "Emergencies & Escalation". Select "Yes", fill `emergency_definition` and `oncall_process`, leave `emergency_script` blank. Click through to Review.

Expected: "Emergencies & Escalation" group shows `Has true emergencies: Yes`, the two filled values, and no row for `emergency_script`. Edit link returns to step 4 with the radio selection and text intact.

- [ ] **Step 4: Commit**

```bash
cd ~/development/steveford-us
git add intake.html
git commit -m "Add Emergencies & Escalation step to intake form"
```

---

### Task 6: Tone & Voice step

**Files:**
- Modify: `intake.html` — replace the `data-step="5"` section's placeholder content, extend `FIELD_GROUPS`

**Interfaces:**
- Consumes: same `FIELD_GROUPS` push pattern.
- Produces: `formData.agent_name_preference`, `formData.tone`, `formData.voice_gender_preference`, `formData.never_say`.

- [ ] **Step 1: Replace the Tone & Voice section's placeholder content**

Find:

```html
      <section class="step" data-step="5">
        <h2>Tone &amp; Voice</h2>
        <p class="step-sub">Added in a later task.</p>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>
```

Replace with:

```html
      <section class="step" data-step="5">
        <h2>Tone &amp; Voice</h2>
        <p class="step-sub">Last stretch — how should your agent sound?</p>

        <label for="agent_name_preference">Agent name preference</label>
        <input type="text" name="agent_name_preference" id="agent_name_preference" placeholder="e.g. Ava, or 'no preference, suggest one'">

        <label>Tone</label>
        <div class="radio-row"><input type="radio" name="tone" value="Warm & casual" id="tone_warm"><label for="tone_warm">Warm &amp; casual</label></div>
        <div class="radio-row"><input type="radio" name="tone" value="Professional & polished" id="tone_pro"><label for="tone_pro">Professional &amp; polished</label></div>
        <div class="radio-row"><input type="radio" name="tone" value="Playful" id="tone_playful"><label for="tone_playful">Playful</label></div>
        <div class="radio-row"><input type="radio" name="tone" value="Formal" id="tone_formal"><label for="tone_formal">Formal</label></div>

        <label>Voice gender preference</label>
        <div class="radio-row"><input type="radio" name="voice_gender_preference" value="Male" id="voice_male"><label for="voice_male">Male</label></div>
        <div class="radio-row"><input type="radio" name="voice_gender_preference" value="Female" id="voice_female"><label for="voice_female">Female</label></div>
        <div class="radio-row"><input type="radio" name="voice_gender_preference" value="No preference" id="voice_none"><label for="voice_none">No preference</label></div>

        <label for="never_say">Anything the agent should never say or do?</label>
        <textarea name="never_say" id="never_say" placeholder="e.g. Never quote exact repair prices. Never argue with an upset caller."></textarea>

        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" onclick="nextStep()">Next</button></div>
      </section>
```

- [ ] **Step 2: Extend `FIELD_GROUPS`**

Append after the `Emergencies & Escalation` group entry added in Task 5:

```js
    { step: 5, title: 'Tone & Voice', fields: [
      { key: 'agent_name_preference', label: 'Agent name preference' },
      { key: 'tone', label: 'Tone' },
      { key: 'voice_gender_preference', label: 'Voice gender preference' },
      { key: 'never_say', label: 'Never say/do' },
    ]},
```

- [ ] **Step 3: Manually verify**

Reload, fill Step 1 required fields, click through to "Tone & Voice". Fill `agent_name_preference`, select "Warm & casual" and "Female", fill `never_say`. Click through to Review.

Expected: "Tone & Voice" group shows all four values correctly (radio selections show their label text, e.g. `Tone: Warm & casual`). Edit link returns to step 5 with all selections/text intact.

At this point every step (0-5) has real fields and Review shows all six groups. Do one full end-to-end pass: fill every field on every step (including at least one transfer row), reach Review, and confirm every single field you entered appears correctly under the right group.

- [ ] **Step 4: Commit**

```bash
cd ~/development/steveford-us
git add intake.html
git commit -m "Add Tone & Voice step to intake form; all six steps now complete"
```

---

### Task 7: Web3Forms submission, honeypot, success state, and failure fallback

**Files:**
- Modify: `intake.html` — add honeypot field, submission JS, success/failure UI

**Interfaces:**
- Consumes: `formData` (from Task 1), `FIELD_GROUPS` (from Tasks 1-6), `escapeHtml(str)` (from Task 1).
- Produces: `WEB3FORMS_ACCESS_KEY` (constant, must be replaced with a real key before this goes live), `buildPlainTextSummary()`, `submitForm()`, `showConfirm()`, `showSubmitError()`.

**Prerequisite (human step, cannot be done by an agent):** Sign up at https://web3forms.com, create an access key for `steveford.us`, and have it ready to paste into the code below.

- [ ] **Step 1: Add the honeypot field to the form**

Find the opening `<form id="intake-form">` tag in `intake.html` and add a hidden honeypot input directly after it:

```html
    <form id="intake-form">
      <input type="checkbox" name="botcheck" id="botcheck" style="display:none" tabindex="-1" autocomplete="off">
```

- [ ] **Step 2: Replace the Review step's Submit button and add confirmation/error containers**

Find:

```html
      <section class="step" data-step="6">
        <h2>Review &amp; Submit</h2>
        <p class="step-sub">Check everything looks right, then send it over.</p>
        <div id="review-content"></div>
        <div class="nav-row"><button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button><div class="spacer"></div><button type="button" class="btn btn-primary" id="submit-btn">Submit</button></div>
      </section>
```

Replace with:

```html
      <section class="step" data-step="6">
        <h2>Review &amp; Submit</h2>
        <p class="step-sub">Check everything looks right, then send it over.</p>
        <div id="review-content"></div>
        <div id="submit-error"></div>
        <div class="nav-row" id="review-nav">
          <button type="button" class="btn btn-ghost" onclick="prevStep()">Back</button>
          <div class="spacer"></div>
          <button type="button" class="btn btn-primary" id="submit-btn" onclick="submitForm()">Submit</button>
        </div>
      </section>

      <section class="step" data-step="7" id="confirm-step">
        <div class="confirm">
          <h2>Thanks — got it!</h2>
          <p>I'll follow up within a business day to talk through the details and put together your agent.</p>
        </div>
      </section>
```

- [ ] **Step 3: Add a `data-step="7"` progress-bar-safe handling note**

`showStep` and the progress dots only need to handle steps 0-6 during normal navigation — step 7 (confirmation) is a terminal state reached only via `submitForm()`, never via Next/Back, so it does not need its own progress dot. No change needed to the `.progress` HTML block.

- [ ] **Step 4: Add submission JS**

Find the end of the script (after the `bindInputs(); showStep(0);` lines added in Task 4) and append:

```js
  const WEB3FORMS_ACCESS_KEY = 'REPLACE_WITH_YOUR_WEB3FORMS_ACCESS_KEY';

  function buildPlainTextSummary() {
    return FIELD_GROUPS.map(group => {
      const lines = group.fields.map(f => {
        let value = formData[f.key];
        if (Array.isArray(value)) value = value.join(', ');
        return value ? `${f.label}: ${value}` : null;
      }).filter(Boolean);
      return `${group.title}\n${lines.join('\n')}`;
    }).join('\n\n');
  }

  function showConfirm() {
    document.querySelectorAll('.step').forEach(sec => sec.classList.remove('active'));
    document.getElementById('confirm-step').classList.add('active');
    document.getElementById('progress').style.display = 'none';
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }

  function showSubmitError() {
    const summary = buildPlainTextSummary();
    const mailBody = encodeURIComponent(summary);
    const mailto = `mailto:steve@steveford.us?subject=${encodeURIComponent('Intake form (manual submit): ' + (formData.business_name || 'Unknown business'))}&body=${mailBody}`;
    document.getElementById('submit-error').innerHTML = `
      <div class="error-box">
        <p>Something went wrong sending this automatically. Your answers aren't lost — copy them or email them directly and I'll get it.</p>
        <button type="button" class="btn btn-ghost" id="copy-answers-btn">Copy my answers</button>
        <a href="${mailto}" class="btn btn-ghost" style="margin-left:10px;text-decoration:none;display:inline-flex">Email my answers</a>
      </div>`;
    document.getElementById('copy-answers-btn').addEventListener('click', () => {
      navigator.clipboard.writeText(summary).then(() => {
        document.getElementById('copy-answers-btn').textContent = 'Copied!';
      });
    });
  }

  async function submitForm() {
    const submitBtn = document.getElementById('submit-btn');
    submitBtn.disabled = true;
    submitBtn.textContent = 'Sending...';
    document.getElementById('submit-error').innerHTML = '';

    const payload = Object.assign({}, formData, {
      access_key: WEB3FORMS_ACCESS_KEY,
      subject: `New intake form: ${formData.business_name || 'Unknown business'}`,
      botcheck: document.getElementById('botcheck').checked,
    });

    try {
      const res = await fetch('https://api.web3forms.com/submit', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      });
      const result = await res.json();
      if (res.ok && result.success) {
        showConfirm();
      } else {
        showSubmitError();
      }
    } catch (err) {
      showSubmitError();
    } finally {
      submitBtn.disabled = false;
      submitBtn.textContent = 'Submit';
    }
  }
```

- [ ] **Step 5: Get a real Web3Forms access key and wire it in**

Sign up at https://web3forms.com, create an access key, then replace `REPLACE_WITH_YOUR_WEB3FORMS_ACCESS_KEY` in the `WEB3FORMS_ACCESS_KEY` constant with the real key.

- [ ] **Step 6: Manually verify success path**

Reload `intake.html`, fill every step (real values, including your own email so you receive the test), reach Review, click Submit.

Expected: button shows "Sending..." briefly, then the page replaces the whole form with "Thanks — got it!", the progress bar disappears, and within a minute an email arrives at the inbox tied to your Web3Forms access key containing all submitted fields.

- [ ] **Step 7: Manually verify failure path**

Temporarily change `WEB3FORMS_ACCESS_KEY` to an obviously invalid value (e.g. `'invalid-test-key'`), reload, fill the form, submit.

Expected: button briefly shows "Sending...", then an error box appears above the nav row with "Copy my answers" and "Email my answers" buttons — the form does NOT silently fail with nothing shown. Click "Copy my answers", then paste elsewhere (e.g. a text editor) to confirm the clipboard contains a readable plain-text summary of every field you entered. Click "Email my answers" and confirm it opens a mail client with a pre-filled subject and body containing the summary.

Restore the real access key afterward.

- [ ] **Step 8: Commit**

```bash
cd ~/development/steveford-us
git add intake.html
git commit -m "Wire Web3Forms submission with honeypot, success confirmation, and failure fallback"
```

---

### Task 8: Mobile responsiveness and full manual QA pass

**Files:**
- Modify: `intake.html` — add a mobile media query if the verify step below finds issues

**Interfaces:**
- None new — this task only adjusts CSS and runs the full checklist from the spec's Testing plan.

- [ ] **Step 1: Add mobile-friendly adjustments**

Find the `.wrap` rule:

```css
  .wrap{max-width:720px;margin:0 auto;padding:48px 24px 96px}
```

Add directly after the closing `</style>`'s existing rules (anywhere in the `<style>` block, e.g. right after the `.error-box` rules at the end):

```css
  @media(max-width:600px){
    header.top{padding:28px 0}
    .card{padding:20px}
    .nav-row{flex-wrap:wrap;gap:10px}
    .review-row{flex-direction:column;gap:2px}
    .review-label{min-width:0}
  }
```

- [ ] **Step 2: Run the full manual QA checklist from the spec**

Using a real browser (resize to ~375px wide, or use device toolbar emulation, for the mobile pass), work through every item:

1. Each step's "Next" is blocked until that step's required fields are filled (only Step 1 has required fields — confirm Steps 2-6 always allow Next with nothing filled).
2. Back/Next preserves previously entered values across every step.
3. Repeatable transfer-destination rows can be added and removed cleanly, including down to zero rows (add two, remove both, confirm `transfer_destinations` is an empty array and Review shows no "Transfer numbers" row).
4. Review screen accurately reflects every field entered across all six steps; every "Edit" link jumps to the correct step.
5. A real test submission arrives at your email via Web3Forms (already verified in Task 7, re-confirm here as part of the full pass).
6. Simulated failure (wrong access key) triggers the fallback instead of failing silently (already verified in Task 7, re-confirm here).
7. At 375px width: progress bar, all form fields, and Next/Back/Submit buttons remain usable with no horizontal scrolling or overlapping text.
8. Open the browser console and confirm there are zero JS errors through a full run-through (fill every step, submit).

Fix anything that fails before proceeding.

- [ ] **Step 3: Commit**

```bash
cd ~/development/steveford-us
git add intake.html
git commit -m "Add mobile responsive styles; complete manual QA pass on intake form"
```
