# Task: Global Search Command Palette

## 1. Issue Description

### Problem and justification

Atomic CRM has no cross-resource search. A user who wants to open a specific record must first guess which resource it belongs to (contact, company, or deal), navigate to that list page, wait for the list to render, then type into the list's filter control. If the guess was wrong, they have to start over. There is no single place to ask "show me anything named 'Acme'" and jump straight to it.

### Motivation

- Lookup is the single most frequent interaction with a CRM. Each extra click compounds across dozens of daily uses.
- Users rarely remember whether a name refers to a person (contact) or an organization (company). Forcing them to pick a resource before typing is the wrong mental model.
- The `Ctrl/Cmd + K` palette pattern is now a near-universal standard in modern SaaS (Linear, GitHub, Slack, Notion). Its absence is a visible product gap.
- One new surface improves perceived speed across the entire app without touching any existing page.

### Overview

Add a **Global Command Palette**: a modal dialog, invokable from anywhere in the authenticated app, that accepts a text query and returns results drawn simultaneously from the **contacts**, **companies**, and **deals** resources. Selecting a result closes the palette and navigates to that record's detail view.

---

## 2. Requirements

### 2.1 Invocation and dismissal

- The palette opens when the user presses `Ctrl + K` on Windows/Linux or `Cmd + K` on macOS, from any authenticated page. The shortcut works regardless of the currently focused element and does not trigger the browser's default behavior for that key combination.
- A visible trigger control appears in the top navigation bar of every authenticated page. Activating it opens the palette. Its label displays the platform-appropriate shortcut hint (`⌘K` on macOS, `Ctrl K` elsewhere).
- The palette closes when the user presses `Escape`, clicks the backdrop outside the dialog, clicks an explicit close control, or activates a result.
- Opening and closing the palette without activating a result does not change the current page's URL or scroll position.

### 2.2 Query and results

- The palette searches **contacts**, **companies**, and **deals** simultaneously.
- Matching is **case-insensitive substring** against:
  - Contacts — first name, last name, email, job title
  - Companies — name
  - Deals — name
- Queries shorter than **2 characters** do not perform a search and do not display results.
- Each resource contributes **at most 5 results** per query; the palette therefore shows at most 15 results total.
- Results are grouped by resource and rendered in a fixed order: **Contacts → Companies → Deals**. Groups with zero results are omitted (no empty group headers).
- Results must settle within approximately **300 ms** after the user stops typing. The palette must not issue a separate network request for every keystroke; a user typing continuously should not see results flicker on each character.
- The palette must be resilient to **stale responses**: if the user types `Ada`, then quickly replaces it with `Grace`, the final visible results must correspond to `Grace`, never to `Ada`, regardless of network ordering.

### 2.3 Selection and navigation

- On each render of results, the **first result is highlighted** by default. When the query changes and new results arrive, the highlight resets to the new first result.
- `ArrowDown` moves the highlight to the next result; `ArrowUp` to the previous.
- `Enter` activates the currently highlighted result. Clicking a result also activates it.
- Hovering a result with the pointer **updates the highlight** but does not activate.
- Activating a result:
  1. closes the palette,
  2. navigates to the record's detail view:
     - Contact → `/contacts/:id/show`
     - Company → `/companies/:id/show`
     - Deal → `/deals/:id/show`

### 2.4 Visible states

The palette must handle and visually distinguish the following states:

| State | Condition |
|---|---|
| **Idle** | Query length < 2 characters. |
| **Loading** | Query ≥ 2 characters and at least one underlying request is in flight. |
| **Populated** | Query ≥ 2 characters and at least one result is available. |
| **Empty** | Query ≥ 2 characters, all underlying requests have resolved, and zero results were returned. |
| **Error** | At least one underlying request failed. Results from successful requests, if any, remain visible alongside the error indication. |

While transitioning between queries, previous results may remain visible to avoid a blank flash, provided stale-response protection (2.2) is preserved.

### 2.5 Accessibility

- The palette is a **modal dialog** with a clear accessible name (e.g., "Global search").
- On open, focus moves to the query input.
- The results list is exposed as a **listbox** with each result as an **option**. The currently highlighted option is communicated through ARIA state; screen-reader users hear which option is active as the highlight moves.
- All behavior described in this document works with keyboard alone — no interaction requires a pointing device.

### 2.6 Out of scope

- Persisting recent searches across reloads or sessions.
- Searching resources other than contacts, companies, and deals (e.g., tasks, notes, tags, sales).
- Fuzzy matching, typo tolerance, or any ranking logic beyond the order in which the data source returns results.
- Inline actions from within the palette (e.g., "Create new contact"). The palette is **read-only**; it only navigates.
- Analytics or telemetry for search queries.

---

## 3. Component Contract and Visual Specification

### 3.1 Structural contract

Nothing is rendered when the palette is closed. When open, the DOM exposes the following elements, identified by accessible role and name (not by tag or class choice):

- A **dialog** with accessible name `"Global search"`.
- Inside the dialog:
  - A **textbox** with accessible name `"Search"` and visible placeholder text along the lines of `"Search contacts, companies, deals…"`. This textbox is focused immediately on open.
  - A **button** with accessible name `"Close"`.
  - A **listbox** that contains the grouped results:
    - Each non-empty group is preceded by a **heading** whose accessible text is exactly `"Contacts"`, `"Companies"`, or `"Deals"`.
    - Each result is an **option** that exposes:
      - A primary visible label (contact full name, company name, or deal name).
      - Where applicable, a secondary label containing contextual information (e.g., a contact's job title and/or company name; a deal's company or stage).
      - A visible resource-type indicator (icon, badge, or equivalent) so that a sighted user can tell contacts, companies, and deals apart even when scrolling.

### 3.2 Invocation trigger

A persistent control in the top navigation of every authenticated page:

- A **button** with accessible name `"Open global search"`.
- Visible content: a search glyph, a label (e.g., `"Search"`), and a keyboard-shortcut hint (`⌘K` on macOS, `Ctrl K` elsewhere) rendered as visibly distinct from the label.
- Clicking the button opens the palette and is fully equivalent to pressing the keyboard shortcut.

### 3.3 Inputs and outputs

The palette is self-contained and takes no props. It is mounted once in the authenticated application shell.

Externally observable effects:

- **Open**: caused by the keyboard shortcut or the invocation trigger.
- **Close**: caused by `Escape`, backdrop click, close button, or result activation.
- **Navigate**: on result activation, the browser URL changes to the target detail route (see §2.3). No other application state is modified by the palette.

The palette reads data through the application's existing data-access layer. It does not receive data or a search function through props.

### 3.4 Behavioral specification

**Keyboard (while open):**

- Typing updates the query.
- `ArrowDown` / `ArrowUp` move the highlight.
- `Enter` activates the highlighted result.
- `Escape` closes the palette.

**Keyboard (while closed):**

- `Ctrl + K` / `Cmd + K` opens the palette, with the default browser behavior for that shortcut suppressed.

**Pointer:**

- Hover on a result updates the highlight only.
- Click on a result activates it.
- Click on the backdrop closes the palette.
- Click on the close button closes the palette.
- Click on the invocation trigger opens the palette.

**Focus management:**

- On open: focus moves to the query input.
- On result activation: the palette closes before navigation occurs; the destination page is responsible for its own focus behavior.

### 3.5 Visual specification

The following layout and style properties must hold. Pixel-exact matching is not required.

- The dialog is **horizontally centered** and positioned in the upper third of the viewport, with a maximum width of approximately **640 px** and enough vertical margin that the dialog never touches the bottom of the viewport.
- The backdrop is a **semi-transparent overlay** that visually dims the underlying page.
- The dialog has a clearly **elevated appearance** — shadow, border, or both — distinguishing it from the backdrop.
- The query input spans the **full width** of the dialog and is visually separated from the results area by a divider.
- Each result row is approximately **40–48 px** tall and contains:
  - a leading resource-type indicator,
  - a primary label,
  - a secondary label in a de-emphasized color (where applicable),
  - an optional trailing visual cue (e.g., a return-key glyph) on the currently highlighted row.
- Group headings are visually de-emphasized relative to result rows (smaller size, muted color, optionally uppercase).
- The highlighted row has a **distinct background** that is clearly different from non-highlighted rows. Exactly one row is highlighted at a time when the list is non-empty.
- The palette renders correctly in both **light and dark themes**, using the theme variables defined by the application.
- On viewport widths below ~640 px, the dialog expands to nearly full width with small lateral margins. All described behavior remains unchanged on narrow viewports.

### 3.6 Acceptance checklist

A solution is acceptable when all of the following are observably true:

- [ ] The palette opens via both `Ctrl/Cmd + K` and the nav trigger, from any authenticated page.
- [ ] The palette closes via `Escape`, backdrop click, close control, and result activation.
- [ ] Queries of fewer than 2 characters show no results.
- [ ] Typing returns grouped results for contacts, companies, and deals within ~300 ms of the user stopping, with no per-keystroke flicker and no stale-response leakage.
- [ ] Each group shows at most 5 results and empty groups are not displayed.
- [ ] The first result is highlighted by default; arrow keys moves the highlight; hover moves the highlight; click and `Enter` activate.
- [ ] Activating a result closes the palette and navigates to the correct detail route.
- [ ] All five states (idle, loading, populated, empty, error) are visually distinguishable.
- [ ] Focus moves to the query input on open.
- [ ] The palette is fully operable with keyboard alone.
- [ ] The palette renders correctly in light and dark themes and on viewports down to ~375 px wide.