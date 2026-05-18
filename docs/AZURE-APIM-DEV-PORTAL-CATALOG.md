# APIM Developer Portal — API Catalog Landing Page

This guide turns the default APIM Developer Portal landing page (Contoso-style hero + marketing copy) into a **catalog dashboard** that lists every API in the APIM instance.

The change is applied **once per APIM instance**, not per API. It is a manual operation on the Azure side — there is no GitHub workflow that does it for you. The same procedure applies to both `sam-apim` (non-live) and the live APIM when you're ready to roll it out there.

| Item                          | Detail                                                                                  |
|-------------------------------|-----------------------------------------------------------------------------------------|
| Target dev portal             | `https://sam-apim.developer.azure-api.net/` (non-live) — same steps for live.            |
| Scope                         | APIM instance level — affects every consumer who visits the portal.                      |
| Reversible?                   | Yes. The portal has revisions; you can publish a previous revision to roll back.         |
| Owner                         | **DevOps** (it's an instance-level Azure operation, not a per-repo change).              |
| Time                          | ~15 minutes.                                                                             |

---

## Prerequisites

- Role on the APIM resource that grants write access to the developer portal: `API Management Service Contributor` is sufficient.
- A browser session signed in to the same Azure account.
- A short outage on the dev portal landing page is OK (no consumers are impacted before "Publish").

---

## Target layout

After this change, the landing page contains, in order:

1. **Header bar** — logo on the left (default), `Sign in` / `Sign up` on the right (default).
2. **Page heading** — single H1: "API Catalog".
3. **List of APIs widget** — every published API in the APIM instance rendered as a card with name, version tag, short description, and a link to its operations page.
4. **Footer** — default.

Removed: Hero ("Explore the APIs"), intro paragraph, "Featured APIs" carousel, marketing testimonial section.

---

## Procedure

### 1. Open the visual editor

1. Sign in to https://portal.azure.com.
2. Navigate to **API Management services** → `sam-apim`.
3. In the left menu under **Developer portal**, select **Portal overview**.
4. Click the **Developer portal** button (top of the blade). This opens a new tab at `https://sam-apim.developer.azure-api.net/?adminUsername=…` with an admin toolbar across the top.

### 2. Switch to edit mode on the home page

1. If the portal opens to anything other than `/`, navigate to home (click the logo or `/` in the URL bar).
2. On the admin toolbar, ensure you are on the **Pages** view and the page selected is **Home** (`/`).
3. Click anywhere on the page canvas to enter the inline editor.

### 3. Remove the default sections

Each default block is a "section" containing one or more "widgets". Working top to bottom:

| Default section          | What it contains                  | Action                                                  |
|--------------------------|-----------------------------------|---------------------------------------------------------|
| Hero                     | Big title + sub-title + CTA button | Click the section outline → trash icon → confirm delete.|
| Intro text               | Paragraph about exploring APIs    | Same.                                                   |
| Featured APIs carousel   | Three card carousel               | Same.                                                   |
| Testimonial / pitch      | Quote / marketing block            | Same.                                                   |

> If your portal version doesn't have all of the above (Azure occasionally tweaks the default), delete whatever you find that isn't the header or footer.

### 4. Add the catalog widget

1. Hover between the header and the footer where the deleted sections used to be. A blue **+ Add section** button appears.
2. Click **+ Add section**. Choose a one-column layout.
3. Inside the new section, click **+ Add widget**.
4. In the widget picker, search for **List of APIs**. Select it.
5. The widget renders inline. Default configuration shows all APIs — leave as-is unless you want to filter by tag.

### 5. Add the page heading

1. Inside the same section, above the widget, click **+ Add widget** again.
2. Choose **Text block**.
3. Set the content to a single line: `# API Catalog` (Markdown H1). Save the widget.
4. Drag the text block above the List-of-APIs widget if it isn't already.

### 6. Save and publish

1. Top toolbar → **Save**. Saves the draft.
2. Top toolbar → **Publish** → confirm in the dialog. This rebuilds the portal and pushes the changes live.
3. Open `https://sam-apim.developer.azure-api.net/` in an incognito window to verify the unauthenticated view.

The publish step usually takes 30–90 seconds. The portal serves the previous version until it completes — no downtime.

---

## Verification

A successful change shows:

- Landing page contains only the **API Catalog** heading and the List-of-APIs widget between the header and footer.
- Every API published in `sam-apim` (currently `api-cp-crime-echo-v1`, plus any others) appears as a card.
- Clicking a card lands on `/api-details/{apiId}` with the rendered spec.
- The dev portal still resolves correctly when not signed in.

If a card you expected is missing, check that the API's **Settings → "Publish on portal"** is enabled (some APIs are unpublished by default).

---

## Rollback

The portal stores every publish as a revision.

1. In the admin editor: **Operations** (left rail) → **Site revisions**.
2. Locate the previous revision (the one immediately before the one you just published).
3. Click **⋯** → **Make current** → **Publish**.

This restores the previous landing page within a minute.

---

## Replicating on the live APIM

When the live APIM is ready to roll out (see [`AZURE-APIM-RELEASE-FLOW.md`](./AZURE-APIM-RELEASE-FLOW.md)), repeat the same procedure on the live instance. There is no spec-style versioning of dev portal content in this repo — the catalog page is configured directly on each APIM instance.

If portal customisation grows beyond a single landing page (custom branding, multiple pages, etc.), consider exporting/importing portal content via the dev portal management API so the configuration becomes scriptable. That's out of scope for v1.

---

## Related documents

- [`AZURE-APIM-PUBLISHING.md`](./AZURE-APIM-PUBLISHING.md) — how this repo publishes the OpenAPI spec into the API that the catalog widget lists.
- [`AZURE-APIM-RELEASE-FLOW.md`](./AZURE-APIM-RELEASE-FLOW.md) — multi-tenant design where this customisation needs to be repeated on the live APIM.
