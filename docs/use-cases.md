# Story to Code: Use cases

Draft, 2026-10-19. Companion to [PRD.md](PRD.md) and [user-stories.md](user-stories.md).

These use cases describe what people do with Story to Code itself (the tool), not with the apps it generates. Each one names the PRD requirements it exercises and the user stories that break it into testable pieces. Where a use case depends on a decision the PRD leaves open, it says which default it assumes; those defaults are collected in [prd-gaps.md](prd-gaps.md).

## Actors

| Actor | Who they are | What they want |
| --- | --- | --- |
| Storyteller | Founder, PM, designer or hobbyist with an app idea (the PRD's primary user) | To see their idea running without writing a spec or code |
| Developer | Takes over the generated scaffold (the PRD's secondary user) | Clean, exportable code plus the stories and App Model that explain it |
| Inference engine | Claude, inferring the App Model from stories | A well-formed story set and current model as input |
| Slice generator | Claude, generating code from App Model diffs | A model diff and the current code for the affected slices |
| External service | Weather, maps or other APIs the generated app calls | Valid keys and requests; may be slow, down or rate-limited |
| Test runner | Runs per-story acceptance tests in the preview and in CI | Deterministic, addressable UI and stable fixtures |

## Use case index

| ID | Use case | Primary actor | PRD refs | Milestone |
| --- | --- | --- | --- | --- |
| UC-01 | Start a project and get a first app from one story | Storyteller | FR-1, FR-4, FR-8, FR-9 | M0 |
| UC-02 | Add a story and watch the app change | Storyteller | FR-1, FR-4, FR-5, FR-9 | M1 |
| UC-03 | Edit, reorder, disable or delete a story | Storyteller | FR-1, FR-7 | M1 |
| UC-04 | Understand a change through its change card | Storyteller | Core loop step 5, FR-3 | M1 |
| UC-05 | Review and resolve assumptions | Storyteller | FR-6 | M3 |
| UC-06 | Pin a decision so later stories cannot overturn it | Storyteller | FR-6, FR-7 | M3 |
| UC-07 | Resolve a contradiction between stories | Storyteller | FR-7 | M1 |
| UC-08 | Survive an identity pivot | Storyteller | Vertical slices: "Big shifts" | M4 |
| UC-09 | Use the generated app inside the preview | Storyteller | FR-9, FR-11 | M1 |
| UC-10 | Switch between wireframe and styled mode | Storyteller | FR-10 | M1 |
| UC-11 | Connect a real external service | Storyteller | FR-12 | M2 |
| UC-12 | Browse history, diff and roll back | Storyteller | FR-13 | M3 |
| UC-13 | Run and read per-story acceptance tests | Storyteller, Test runner | Vertical slices: "Each slice ships with a check" | M3 |
| UC-14 | Target a native store (App Store) | Storyteller | Platform inference | M4 |
| UC-15 | Export to a folder or Git repository | Developer | FR-14 | M5 |
| UC-16 | Reopen a saved project | Storyteller | Implied by FR-13, not stated | M1 |
| UC-17 | Recover when inference or generation fails | Storyteller | Not covered by the PRD | M1 |

## Use cases

### UC-01 Start a project and get a first app from one story

**Goal.** A first visible app within one story (PRD goal 1).
**Preconditions.** The tool is open; no project exists.
**Trigger.** The Storyteller types a first story, such as "Joe logs into an app."

**Main flow**

1. The Storyteller creates a project (or the tool creates an untitled one on first story).
2. The Storyteller enters the first story in free text and submits it.
3. The tool shows a working indicator on the story and in the preview.
4. The inference engine produces App Model v1: default platform mobile, one persona, auth needed.
5. The slice generator produces the login slice: `<login-form>`, an empty home screen, and a minimal data layer.
6. The preview renders a phone frame with the login screen.
7. A change card explains "Created a mobile app with a login screen because Joe logs in."

**Alternate flows**

- 2a. The story is empty or only whitespace: submit is disabled.
- 2b. The story is not about an app at all ("hello"): the tool still creates the generic shell and records a low-confidence assumption, rather than refusing.
- 4a. Inference fails: see UC-17.

**Postconditions.** App Model v1 and code snapshot v1 exist; the story's trace lists the screens, components and entities it produced.

### UC-02 Add a story and watch the app change

**Goal.** Every story change triggers re-inference and a fresh render with a diff (PRD goal 2).
**Preconditions.** A project with at least one story and a rendered app.

**Main flow**

1. The Storyteller adds a story ("Joe is sitting at his desk using his desktop").
2. The inference engine re-reads every enabled story plus pins and confirmed or rejected assumptions, and patches the App Model.
3. The slice generator diffs the model and edits only the affected slices.
4. The preview hot-swaps the changed modules without a full reload and keeps the current screen and form input where possible.
5. New device frames appear if the platform set changed (phone and desktop side by side).
6. A change card lists what was added, changed and removed, and names the causing story.

**Alternate flows**

- 1a. The Storyteller adds a story while a previous change is still processing: the new story is queued and processed after the current one, or the in-flight job is cancelled and restarted with the full set (the PRD does not say which; see gaps).
- 3a. The model diff is empty (the story adds nothing new): no code changes; the change card says the story is already satisfied and links the components that satisfy it.

**Postconditions.** A new version exists; untouched slices are byte-for-byte identical to the previous version.

### UC-03 Edit, reorder, disable or delete a story

**Preconditions.** A project with several stories.

**Main flow**

1. The Storyteller edits the text of an existing story, drags it to a new position, toggles it off, or deletes it.
2. The tool re-infers from the new story set as in UC-02.
3. Artifacts traced only to a deleted or disabled story are removed; artifacts shared with other stories stay.
4. The change card says which story change caused each removal.

**Alternate flows**

- 1a. Delete asks for confirmation, or offers undo, since it can remove screens.
- 2a. Reordering changes which story is "newest" for contradiction handling (FR-7), so it can change the app even when no text changed.

### UC-04 Understand a change through its change card

**Main flow**

1. After each change, a change card appears next to the preview.
2. The card lists added, changed and removed screens, components, entities, integrations and assumptions.
3. Each item names the story that caused it; clicking the item highlights it in the preview and the story in the list.
4. Pivots are labelled as pivots (UC-08).
5. The Storyteller can expand the card into a model diff and a code diff.

### UC-05 Review and resolve assumptions

**Main flow**

1. The assumptions panel lists every inferred guess with a confidence level and its source story ("Sally's app is probably a shopping list, 40%").
2. The Storyteller confirms, rejects, or pins an assumption, or replaces it with their own text.
3. Rejecting an assumption triggers re-inference that must not reintroduce it.
4. Confirming raises it to a fact the engine treats like a story.

**Alternate flows**

- 2a. The Storyteller ignores assumptions: they stay open and the engine may revise them as new stories arrive.

### UC-06 Pin a decision

**Main flow**

1. The Storyteller pins a decision, either from the assumptions panel, from a change card item, or by writing it ("keep it a web app").
2. The pin appears in a pins list.
3. A later story that contradicts the pin does not overturn it; the change card says the story was partly ignored because of the pin and offers to unpin.

### UC-07 Resolve a contradiction between stories

**Main flow**

1. The Storyteller adds a story that contradicts an earlier one ("Joe doesn't need to log in").
2. With no pin involved, the newest story wins (FR-7): the login slice is removed.
3. The change card names both stories and says which one won.
4. The older story is marked as superseded in the list, and the Storyteller can edit, disable or delete it.

### UC-08 Survive an identity pivot

**Main flow**

1. The Storyteller writes a story that changes what the app is ("This app should be a game, and I want it on the App Store").
2. The engine detects a pivot and regenerates freely.
3. The change card is marked as a pivot, summarises the before and after, and offers a one-click rollback to the pre-pivot version.
4. Earlier stories that no longer apply are flagged rather than silently dropped.

**Open point.** What happens to Susie's location, search and saved places after the game pivot is undefined; see gaps.

### UC-09 Use the generated app inside the preview

**Main flow**

1. The Storyteller clicks, types and navigates inside the device frame as if using the real app.
2. Data they enter persists in the preview's IndexedDB and survives a hot-swap.
3. Seed data appears where entities exist but nothing has been entered yet.
4. Browser permission prompts (geolocation) are shown inside the sandbox and the Storyteller can deny them.

**Alternate flows**

- 2a. A model change alters an entity's shape: existing preview data is migrated, or reset with a notice (undefined in the PRD; see gaps).

### UC-10 Switch between wireframe and styled mode

1. The Storyteller toggles wireframe (grayscale boxes and labels) or styled mode.
2. The preview re-renders without re-running inference or generation, and app state is kept.

### UC-11 Connect a real external service

**Main flow**

1. The generated app starts with mocked services (FR-12), and the preview marks mocked data as such.
2. The Storyteller opens the integrations panel, picks a provider and enters a key.
3. The tool swaps the mock adapter for the live one; no code outside the adapter changes.
4. The key is stored outside the generated code and excluded from exports.

**Alternate flows**

- 3a. The key is invalid or the service fails: the adapter falls back to the mock and shows the error.

### UC-12 Browse history, diff and roll back

1. The Storyteller opens history and sees one version per story change, labelled with the causing change.
2. They pick any two versions and see the model diff, code diff and a side-by-side preview.
3. They roll back; the story set, App Model, pins and code all return to that version, and the rollback itself becomes a new version (nothing is lost).

### UC-13 Run and read per-story acceptance tests

1. Each slice ships with a generated acceptance test tied to its story.
2. Tests run automatically after every change, in the preview.
3. Each story shows pass, fail or not yet run.
4. A story whose test fails after a later change is flagged, and the change card says which later story broke it.

### UC-14 Target a native store

1. A story mentions the App Store or an iPhone app.
2. The UI is restyled to Apple HIG, a Capacitor project is added to the output, and an App Store readiness checklist appears.
3. Each checklist item says whether it is done, generated, or needs the Storyteller (account, icons, privacy text).

### UC-15 Export to a folder or Git repository

1. The Developer chooses export to folder (zip download) or to a Git repository.
2. The export contains the generated app, `stories.md` (or JSON), the App Model, the acceptance tests and a README explaining how to run them.
3. The exported app runs by opening `index.html` through any static server, with no build step.
4. No secrets are present in the export.

### UC-16 Reopen a saved project

1. The Storyteller closes the tool and comes back later.
2. Their projects are listed; opening one restores stories, model, pins, history and preview data.

### UC-17 Recover when inference or generation fails

1. A Claude call times out, returns invalid structured output, or the generated code throws in the preview.
2. The tool keeps the last good version on screen, marks the story as failed with a short reason, and offers retry.
3. Generated code that fails to load or fails its own story's test is not promoted to the current version.
