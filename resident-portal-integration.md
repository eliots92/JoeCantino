# Resident Portal → Maintenance Reports integration

The resident-facing portal (built on a separate website) connects to this system by
POSTing maintenance requests to a **dedicated Formspree form**, keeping resident
requests in their own inbox/notification stream alongside the field app's forms.

- **Resident Requests endpoint:** `https://formspree.io/f/mzdlkrrn` ← the resident site POSTs here
- Field app reports → `mjgqaald` (`source: "field-app"`)
- Field app invoices → `mlgyppve`
- Resident submissions are tagged `source: "resident-portal"`

## Escalation model

Only **escalated** requests reach this pipeline — tenants are shown quick self-help
checks for their issue first, and the request submits only if those didn't solve it.
Every submission therefore carries:

- `escalated: yes` — the tenant went through triage and the problem persists
- `triedSteps` — which self-help checks the tenant performed, so nobody re-does the basics

**Exception:** anything marked **Emergency** skips triage entirely and submits
immediately — never put self-help between a tenant and reporting a gas smell,
active flooding, or sparking outlet.

## Field contract

Use these exact field names so resident requests read the same as field reports:

| Field | Value |
| --- | --- |
| `_subject` | Email subject, e.g. `Escalated request — Plumbing @ 1051 2nd Ave S` |
| `type` | `report` |
| `source` | `resident-portal` |
| `escalated` | `yes` (self-help attempted) or `emergency-bypass` |
| `triedSteps` | Semicolon-separated list of self-help checks performed, e.g. `Plunged the toilet; Checked the shutoff valve` |
| `issue` | What's wrong, e.g. `Plumbing`, `Heating / Cooling`, `Electrical`, `Pest`, `Other` |
| `category` | One of: `Safety & Emergency`, `Building Systems`, `Structure & Interior`, `Exterior & Grounds`, `Sanitation & Pest`, `Utilities & General` |
| `urgency` | `emergency`, `urgent`, `soon`, or `routine` |
| `property` | `1051 2nd Ave S` or `Thompson's Station` |
| `unitArea` | Unit / area, e.g. `Unit B, kitchen` |
| `note` | Resident's description |
| `contactName` / `contactPhone` / `contactEmail` | Resident contact info |
| `status` | `new` |
| `photo1`, `photo2`, … | Image files (send as `multipart/form-data`) |

Requests without photos can be sent as JSON (`Content-Type: application/json`,
`Accept: application/json`). Requests with photos must be `multipart/form-data`
(don't set Content-Type manually — the browser sets the boundary).

## Drop-in snippet (with triage gate)

A self-contained request form for the resident site. Picking an issue reveals its
self-help checklist; the submit button unlocks only after the tenant confirms the
checks didn't fix it (or selects Emergency). Paste and restyle as needed.

```html
<form id="maint-form">
  <label>Property
    <select name="property" required>
      <option>1051 2nd Ave S</option>
      <option>Thompson's Station</option>
    </select>
  </label>
  <label>Unit / area <input name="unitArea" placeholder="e.g. Unit B, kitchen"></label>
  <label>Issue
    <select name="issue" required>
      <option value="">— pick one —</option>
      <option>Plumbing</option><option>Heating / Cooling</option>
      <option>Electrical</option><option>Appliance</option>
      <option>Pest</option><option>Doors / locks</option><option>Other</option>
    </select>
  </label>
  <label>How urgent?
    <select name="urgency" required>
      <option value="emergency">Emergency — safety risk right now</option>
      <option value="urgent">Urgent — today</option>
      <option value="soon" selected>Soon — this week</option>
      <option value="routine">Routine — whenever</option>
    </select>
  </label>

  <fieldset id="triage" hidden>
    <legend>Before we send this — quick checks</legend>
    <div id="triage-steps"></div><!-- checklist injected per issue -->
    <p>
      <button type="button" id="triage-fixed">That fixed it 🎉</button>
      <label><input type="checkbox" id="triage-stuck"> I tried these — still not working</label>
    </p>
  </fieldset>
  <p id="triage-thanks" hidden>Glad that sorted it — no request needed. Feel free to close this page.</p>

  <label>Describe the problem <textarea name="note" required></textarea></label>
  <label>Your name <input name="contactName" required></label>
  <label>Phone <input name="contactPhone" type="tel"></label>
  <label>Email <input name="contactEmail" type="email"></label>
  <label>Photos <input name="photos" type="file" accept="image/*" multiple></label>
  <button type="submit" id="maint-submit" disabled>Submit request</button>
  <p id="maint-status"></p>
</form>

<script>
// Per-issue self-help checklists. Edit freely — each string becomes a checkbox,
// and checked items are reported to maintenance as `triedSteps`.
const SELF_HELP = {
  "Plumbing": [
    "Toilet: plunged it for 20–30 seconds",
    "Clog: tried a sink plunger / removed visible debris from the drain",
    "Leak: turned the fixture's shutoff valve (under sink / behind toilet) clockwise",
    "Low hot water: checked other faucets to see if it's just one fixture"
  ],
  "Heating / Cooling": [
    "Checked the thermostat is on the right mode (heat/cool) and set 5° past room temp",
    "Replaced the thermostat batteries",
    "Checked the air filter isn't clogged",
    "Checked the breaker for the HVAC unit"
  ],
  "Electrical": [
    "Reset the breaker (flip fully OFF, then ON)",
    "Pressed RESET on the GFCI outlet (bathroom/kitchen outlets often chain together)",
    "Tried a different device in the outlet to rule out the device"
  ],
  "Appliance": [
    "Checked it's plugged in and the outlet works",
    "Checked the breaker",
    "Garbage disposal: pressed the red reset button underneath"
  ],
  "Pest": [
    "Removed food sources / sealed trash",
    "Noted where and when you're seeing them (helps treatment)"
  ],
  "Doors / locks": [
    "Checked for anything blocking the latch / strike plate",
    "Tried the spare key if you have one"
  ],
  "Other": []
};

const form = document.getElementById("maint-form");
const triage = document.getElementById("triage");
const stepsBox = document.getElementById("triage-steps");
const stuck = document.getElementById("triage-stuck");
const submitBtn = document.getElementById("maint-submit");
const thanks = document.getElementById("triage-thanks");

function refreshGate(){
  const emergency = form.elements.urgency.value === "emergency";
  const issue = form.elements.issue.value;
  const steps = SELF_HELP[issue] || [];
  triage.hidden = emergency || !issue || !steps.length;
  // Emergencies bypass triage; otherwise require the "still not working" confirmation.
  submitBtn.disabled = !emergency && !(issue && (!steps.length || stuck.checked));
}
form.elements.issue.addEventListener("change", () => {
  const steps = SELF_HELP[form.elements.issue.value] || [];
  stepsBox.innerHTML = steps.map(s =>
    `<label><input type="checkbox" class="tried" value="${s}"> ${s}</label>`).join("");
  stuck.checked = false; thanks.hidden = true;
  refreshGate();
});
form.elements.urgency.addEventListener("change", refreshGate);
stuck.addEventListener("change", refreshGate);
document.getElementById("triage-fixed").addEventListener("click", () => {
  triage.hidden = true; thanks.hidden = false; submitBtn.disabled = true;
});

form.addEventListener("submit", async (e) => {
  e.preventDefault();
  const status = document.getElementById("maint-status");
  const emergency = form.elements.urgency.value === "emergency";
  const tried = [...form.querySelectorAll(".tried:checked")].map(c => c.value).join("; ");
  const fd = new FormData();
  ["property","unitArea","issue","urgency","note","contactName","contactPhone","contactEmail"]
    .forEach(k => fd.append(k, form.elements[k].value));
  fd.append("type", "report");
  fd.append("source", "resident-portal");
  fd.append("status", "new");
  fd.append("escalated", emergency ? "emergency-bypass" : "yes");
  fd.append("triedSteps", tried || "(none checked)");
  fd.append("_subject", `${emergency ? "EMERGENCY" : "Escalated request"} — ${form.elements.issue.value} @ ${form.elements.property.value}`);
  [...form.elements.photos.files].forEach((f, i) => fd.append("photo" + (i + 1), f));
  status.textContent = "Sending…";
  try {
    const res = await fetch("https://formspree.io/f/mzdlkrrn", {
      method: "POST", headers: { "Accept": "application/json" }, body: fd
    });
    if (!res.ok) throw new Error();
    form.reset();
    status.textContent = "Request received — we're on it.";
    refreshGate();
  } catch {
    status.textContent = "Couldn't send — please try again or call us.";
  }
});
refreshGate();
</script>
```

## Notes

- Formspree accepts cross-origin AJAX submissions, so this works from any domain.
- Photo uploads count against the Formspree plan's file limits; consider downscaling
  client-side if residents send full-res phone photos (the field app does this).
