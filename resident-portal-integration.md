# Resident Portal → Maintenance Reports integration

The resident-facing portal (built on a separate website) connects to this system by
POSTing maintenance requests to the **same Formspree form** the field app uses. Both
sources then land in one inbox/dashboard, distinguishable by the `source` field.

- **Endpoint:** `https://formspree.io/f/mjgqaald` (Maintenance Reports)
- **Field app submissions:** `source: "field-app"`
- **Resident submissions:** `source: "resident-portal"`

## Field contract

Use these exact field names so resident requests read the same as field reports:

| Field | Value |
| --- | --- |
| `_subject` | Email subject, e.g. `Resident request — Plumbing @ 1051 2nd Ave S` |
| `type` | `report` |
| `source` | `resident-portal` |
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

## Drop-in snippet

A self-contained request form for the resident site — paste and restyle as needed:

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
  <label>Describe the problem <textarea name="note" required></textarea></label>
  <label>Your name <input name="contactName" required></label>
  <label>Phone <input name="contactPhone" type="tel"></label>
  <label>Email <input name="contactEmail" type="email"></label>
  <label>Photos <input name="photos" type="file" accept="image/*" multiple></label>
  <button type="submit">Submit request</button>
  <p id="maint-status"></p>
</form>

<script>
document.getElementById("maint-form").addEventListener("submit", async (e) => {
  e.preventDefault();
  const form = e.target, status = document.getElementById("maint-status");
  const fd = new FormData();
  ["property","unitArea","issue","urgency","note","contactName","contactPhone","contactEmail"]
    .forEach(k => fd.append(k, form.elements[k].value));
  fd.append("type", "report");
  fd.append("source", "resident-portal");
  fd.append("status", "new");
  fd.append("_subject", `Resident request — ${form.elements.issue.value} @ ${form.elements.property.value}`);
  [...form.elements.photos.files].forEach((f, i) => fd.append("photo" + (i + 1), f));
  status.textContent = "Sending…";
  try {
    const res = await fetch("https://formspree.io/f/mjgqaald", {
      method: "POST", headers: { "Accept": "application/json" }, body: fd
    });
    if (!res.ok) throw new Error();
    form.reset();
    status.textContent = "Request received — we're on it.";
  } catch {
    status.textContent = "Couldn't send — please try again or call us.";
  }
});
</script>
```

## Notes

- Formspree accepts cross-origin AJAX submissions, so this works from any domain.
- If resident volume grows, consider a third dedicated form (e.g. "Resident Requests")
  so counts and notifications stay separate — the only change is the endpoint URL here.
- Photo uploads count against the Formspree plan's file limits; consider downscaling
  client-side if residents send full-res phone photos (the field app does this).
