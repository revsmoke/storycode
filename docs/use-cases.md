# Story to Code: Use cases

Draft, 2026-10-19. Companion to [PRD.md](PRD.md) and [user-stories.md](user-stories.md).

These use cases describe what people do with Story to Code itself (the tool), not with the apps it generates. Each one names the PRD requirements and sections it exercises; [user-stories.md](user-stories.md) breaks them into testable pieces. They follow the PRD as revised by the gap review (FR-15 to FR-19, the App Model schema, story precedence, data layer, preview sandbox, acceptance tests, export and testing sections).

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
| UC-01 | Start a project and get a first app from one story | Storyteller | FR-1, FR-4, FR-8, FR-9; Stories and precedence | M0 |
| UC-02 | Add a story and watch the app change | Storyteller | FR-1, FR-4, FR-5, FR-9, FR-17 | M1 |
| UC-03 | Edit, reorder, disable or delete a story | Storyteller | FR-1, FR-7; Stories and precedence | M1 |
| UC-04 | Understand a change through its change card | Storyteller | FR-3, FR-16 | M1 |
| UC-05 | Review and resolve assumptions | Storyteller | FR-6, FR-15 | M3 |
| UC-06 | Pin a decision so later stories cannot overturn it | Storyteller | FR-6, FR-7 | M3 |
| UC-07 | Resolve a contradiction between stories | Storyteller | FR-7 | M1 |
| UC-08 | Survive an identity pivot | Storyteller | FR-16; Vertical slices: "Big shifts" | M4 |
| UC-09 | Use the generated app inside the preview | Storyteller | FR-9, FR-11; Data layer; Preview sandbox | M1 |
| UC-10 | Switch between wireframe and styled mode | Storyteller | FR-10 | M1 |
| UC-11 | Connect a real external service | Storyteller | FR-12, FR-19; Preview sandbox and keys | M2 |
| UC-12 | Browse versions, diff and roll back | Storyteller | FR-13; Versions | M3 |
| UC-13 | Run and read per-story acceptance tests | Storyteller, Test runner | Acceptance tests | M3 |
| UC-14 | Target a native store (App Store) | Storyteller | Platform inference | M4 |
| UC-15 | Export to a folder or Git repository | Developer | FR-14; Export | M5 |
| UC-16 | Reopen a saved project | Storyteller | Versions: projects persist | M1 |
| UC-17 | Recover when inference or generation fails | Storyteller | FR-18; Inference contract: validation | M1 |

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

- 2a. The story is empty, only whitespace, or over 2,000 characters: submit is disabled and the limit is shown.
- 2b. The story is not about an app at all ("hello"): the tool still creates the generic shell and records a low-confidence assumption, rather than refusing. The first story always runs with the dial on wild, since there is nothing to keep.
- 4a. Inference fails: see UC-17.

**Postconditions.** App Model v1 and code snapshot v1 exist; the story's trace lists the screens, components and entities it produced.

### UC-02 Add a story and watch the app change

**Goal.** Every story change triggers re-inference and a fresh render with a diff (PRD goal 2).
**Preconditions.** A project with at least one story and a rendered app.

**Main flow**

1. The Storyteller adds a story ("Joe is sitting at his desk using his desktop").
2. The inference engine re-reads every enabled story in order, plus pins and rejected assumptions, and returns a JSON Patch to the App Model, which is validated against the schema and the pins.
3. The slice generator diffs the model and edits only the affected slices.
4. The preview hot-swaps the changed modules without a full reload and keeps the current screen and form input where possible.
5. New device frames appear if the platform set changed (phone and desktop side by side).
6. A change card lists what was added, changed and removed, and names the causing story.

**Alternate flows**

- 1a. The Storyteller adds or edits a story while a previous change is still processing: the in-flight change is cancelled and restarts with the full set after a short debounce, and the preview keeps the last good version meanwhile (FR-17).
- 3a. The model diff is empty (the story adds nothing new): no code changes; a no-op change card names the artifacts that already satisfy it.
- 3b. The story is too vague to change the app ("Joe is happy"): no slice; the change card says so and suggests what the story could add (a place, a device, an action).
- 3c. The story only describes a persona ("Joe is a store manager"): a fact is added to that persona and no screen is produced.

**Postconditions.** A new version exists; untouched slices are byte-for-byte identical to the previous version.

### UC-03 Edit, reorder, disable or delete a story

**Preconditions.** A project with several stories.

**Main flow**

1. The Storyteller edits the text of an existing story, drags it to a new position, toggles it off, or deletes it.
2. The tool re-infers from the new story set as in UC-02.
3. Artifacts traced only to a deleted or disabled story are removed; artifacts shared with other stories stay.
4. The change card says which story change caused each removal.

**Alternate flows**

- 1a. Delete can be undone until the next change, and the deletion is recorded in history.
- 1b. Editing or moving a story keeps its stable id, so traces and change cards survive.
- 2a. The story lower in the list wins a conflict (FR-7), so reordering can change the app even when no text changed.

### UC-04 Understand a change through its change card

**Main flow**

1. After each change, a change card appears next to the preview.
2. The card holds a title, the causing stories, a model diff summary, the files touched, new assumptions, inferred acceptance criteria and test results (FR-16).
3. Each item names the story that caused it; clicking the item highlights it in the preview and the story in the list.
4. Pivots are labelled as pivots (UC-08).
5. The Storyteller can open the diff or roll back from the card. Past cards are listed newest first in the Changes pane.

### UC-05 Review and resolve assumptions

**Main flow**

1. The Assumptions pane lists every open guess with a confidence level (low, medium or high) and its source story ("Sally's app is probably a shopping list", low).
2. The Storyteller confirms, rejects, or pins an assumption.
3. Rejecting adds it to the model's `rejected` list and re-runs inference, which must not make the same guess again.
4. Confirming raises its confidence and removes it from the pane.
5. Pinning adds a pin at the assumption's model path.

**Alternate flows**

- 2a. The Storyteller ignores assumptions: they stay open and the engine may revise them as new stories arrive.

### UC-06 Pin a decision

**Main flow**

1. The Storyteller pins a decision from the Assumptions pane, or by writing it ("keep it a web app").
2. The pin is stored as a JSON Pointer path and value, and is listed in the Assumptions pane where it can be removed.
3. A later story that contradicts the pin does not overturn it: the validator rejects any patch that changes a pinned path, and the change card names the pin that blocked it and offers to unpin and retry.

### UC-07 Resolve a contradiction between stories

**Main flow**

1. The Storyteller adds a story that contradicts an earlier one ("Joe doesn't need to log in").
2. With no pin involved, the story lower in the list wins (FR-7); new stories are appended, so here the newest wins and the login slice is removed.
3. The change card names both stories and says which one won.
4. The older story is marked superseded, naming the story that overrode it. Moving it below the winner makes it win again.

### UC-08 Survive an identity pivot

**Main flow**

1. The Storyteller writes a story that changes what the app is ("This app should be a game, and I want it on the App Store").
2. The engine detects a pivot and regenerates freely.
3. The change card is marked as a pivot, summarises the before and after, and offers a one-click rollback to the pre-pivot version.
4. Earlier stories the pivot no longer satisfies (Susie's location, search and saved places) are flagged on the card and in the list rather than silently dropped; the Storyteller can disable them, rewrite them, or pin what they imply.

### UC-09 Use the generated app inside the preview

**Main flow**

1. The Storyteller clicks, types and navigates inside the device frame as if using the real app.
2. Data they enter persists in the preview's IndexedDB and survives a hot-swap.
3. Seed data appears where entities exist but nothing has been entered yet; it can be cleared from Settings.
4. Geolocation is allowed in the iframe only when the model asks for location, and the Storyteller can deny the prompt.

**Alternate flows**

- 2a. A model change alters an entity's shape: a generated migration runs. Added fields get a default, renamed fields are mapped, and removed fields are copied to an `_archive` store rather than dropped.

### UC-10 Switch between wireframe and styled mode

1. The Storyteller toggles wireframe (grayscale boxes and labels) or styled mode.
2. The preview re-renders without re-running inference or generation, and app state is kept.

### UC-11 Connect a real external service

**Main flow**

1. The generated app starts with mocked services (FR-12), and the preview marks mocked data as such.
2. The Storyteller opens Settings, picks a provider and enters a key.
3. The live adapter is used without regenerating code: in the preview, adapters call the tool's proxy (`/proxy/<integration-id>`), which adds the key.
4. The key never appears in generated code, the App Model or exports; an export reads keys from a `config.js` excluded by `.gitignore`.

**Alternate flows**

- 3a. The key is invalid or the service fails: the adapter falls back to the mock and shows the error (assumed; the PRD does not name the fallback).

### UC-12 Browse versions, diff and roll back

1. The Storyteller opens the Versions pane and sees a timeline with one version per successful change, labelled with its change card title.
2. They pick any two versions and see the model diff, code diff and a side-by-side preview.
3. They roll back; the story list, App Model, pins, assumption answers and code all return to that version, and the rollback itself becomes a new version, so it can be rolled back too.

### UC-13 Run and read per-story acceptance tests

1. Each slice ships with `tests/<story-id>.test.js`, written from the story's acceptance criteria, or from criteria Claude infers and lists on the change card for the Storyteller to correct.
2. All tests run after every change in the preview's built-in runner, including an axe accessibility check on each screen visited.
3. Each story shows pass, fail or not yet run.
4. A version whose own story's test fails is not promoted.
5. An earlier story's test that fails is shown as a regression on that story, and the change card offers a repair scoped to that slice.

### UC-14 Target a native store

1. A story mentions the App Store or an iPhone app.
2. The UI is restyled to Apple HIG, a Capacitor project is added to the output, and an App Store readiness checklist appears.
3. Each checklist item says whether it is done, generated, or needs the Storyteller (account, icons, privacy text).

### UC-15 Export to a folder or Git repository

1. The Developer chooses export to folder (zip download) or to a Git repository.
2. The export contains the `app/` folder (with `stories.json`, `app-model.json` and the tests as Playwright specs), a README on running it and supplying keys in `config.js`, and a `CHANGES.md` built from the change cards. A native target adds `native/` and its App Store checklist.
3. The exported app runs from any static server, with no build step.
4. No secrets are present in the export.
5. Git export makes one commit per version with the change card title as the message.
6. Hand edits to exported code are not read back; the README names this as the eject point.

### UC-16 Reopen a saved project

1. The Storyteller closes the tool and comes back later.
2. Their projects are listed; opening one restores stories, versions, pins, settings and the preview's IndexedDB data exactly as left.

### UC-17 Recover when inference or generation fails

1. A Claude call times out, returns invalid structured output, or the generated code throws in the preview.
2. An invalid patch is sent back to Claude once with the error; on a second failure the change is abandoned.
3. The preview never blanks: the last good version stays up and an error card explains and offers retry (FR-18).
4. Generated code that fails to load or fails its own story's test is not promoted.
