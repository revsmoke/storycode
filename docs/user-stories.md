# Story to Code: User stories and acceptance criteria

Draft, 2026-10-19. Companion to [PRD.md](PRD.md) and [use-cases.md](use-cases.md).

These stories describe Story to Code itself. Each one has acceptance criteria written as Given / When / Then so they can become automated tests later without rewording. Every story is tagged with the test level it belongs to, the milestone it lands in, and the PRD requirement and use case it traces to.

## How these become tests

Story to Code has a non-deterministic core (Claude) wrapped in deterministic plumbing. The test plan splits along that line.

| Level | Tag | What it checks | Runs against | Where |
| --- | --- | --- | --- | --- |
| Unit | `unit` | Pure logic: App Model schema validation, model diffing, version store, trace bookkeeping, pin and contradiction rules | Plain JS modules | Node test runner, every commit |
| Contract | `contract` | Claude responses parse into a valid App Model or code patch; malformed output is rejected | Recorded fixtures plus a schema | Every commit |
| Browser | `browser` | The tool's UI and the generated app behave as described | Playwright with the **fixture engine** (below) | Every commit |
| Eval | `eval` | Inference quality: does story X actually produce the expected model? | The live Claude API, scored against expected model facts | Nightly or on prompt changes; reports a pass rate, not a hard gate |
| Generated | `generated` | The per-story acceptance tests Story to Code writes for the user's app | The generated app in the preview iframe | Inside the product (UC-13) and in exports |

**The fixture engine.** Browser tests must not call Claude. The inference and generation calls sit behind one interface (`engine.infer(storySet, model)` and `engine.generate(modelDiff, files)`), and tests swap in a fixture engine that replays recorded responses keyed by a hash of the story set. The two PRD worked examples (Joe and Sally, Susie) are the first fixture sets. Recording a fixture is a single command that runs the live engine once and saves the responses. Eval tests use the same story sets with the live engine, so a fixture and an eval case are the same scenario at two levels.

**Selectors.** Browser tests address UI by `data-testid`, never by text or CSS. The tool reserves the names in the table at the end of this file. Generated apps get `data-testid` on every custom element root, derived from the App Model id of the screen or component (`screen-login`, `cmp-login-form`), so generated acceptance tests stay stable across restyles and wireframe mode.

**Time and network.** Browser tests stub geolocation, freeze the clock, and route external APIs to the mock adapters (FR-12), so the Susie scenario is reproducible.

## Story format

Each story uses the PRD's optional structure (FR-2): *As a … I want … so that …*, followed by acceptance criteria. IDs are stable; add new stories at the end of their epic rather than renumbering.

---

## Epic A: Stories (writing and managing them)

### US-A1 Write a story in free text
*As a Storyteller, I want to type a story in plain language, so that I can describe my idea without learning a format.*
Tags: `browser` · M0 · FR-1 · UC-01

- **Given** an empty project, **when** I type "Joe logs into an app." and submit, **then** the story appears in the list as story 1 and processing starts.
- **Given** the story input is empty or whitespace, **then** submit is disabled.
- **Given** a story of 2,000 characters, **when** I submit it, **then** it is accepted in full (the limit is configurable; see gaps).
- **Given** I submit with the keyboard (Enter, or Cmd/Ctrl+Enter in a multi-line input), **then** it behaves the same as the submit button.

### US-A2 Optionally use structured form
*As a Storyteller who knows the format, I want to write "As a…, I want…, so that…" and add acceptance criteria, so that the engine gets more precise input.*
Tags: `unit`, `browser` · M1 · FR-2

- **Given** a story in structured form, **when** it is parsed, **then** persona, want and reason are stored as separate fields and shown as such.
- **Given** a story with acceptance criteria, **then** each criterion becomes an assertion in that story's generated acceptance test (US-H1).
- **Given** a free-text story, **then** no structure is required and no warning is shown.

### US-A3 Edit a story
*As a Storyteller, I want to edit a story I already wrote, so that I can refine the idea.*
Tags: `browser` · M1 · FR-1 · UC-03

- **Given** story 2 exists, **when** I edit its text and save, **then** re-inference runs and a new version is recorded with cause "edited story 2".
- **Given** I start editing and cancel, **then** nothing changes and no version is recorded.

### US-A4 Reorder stories
*As a Storyteller, I want to drag stories into a different order, so that I can control which one counts as newest.*
Tags: `browser`, `unit` · M1 · FR-1, FR-7 · UC-03

- **Given** stories 1 to 3, **when** I drag story 3 above story 1, **then** the list order and the stored order both change and re-inference runs.
- **Given** two contradictory stories with no pin, **when** I swap their order, **then** the winning story swaps too (unit test on the contradiction rule).
- Reordering is also possible with the keyboard (accessibility).

### US-A5 Disable and re-enable a story
*As a Storyteller, I want to switch a story off without deleting it, so that I can try the app without it.*
Tags: `browser` · M1 · FR-1 · UC-03

- **Given** story 3 is enabled, **when** I disable it, **then** it is visibly dimmed, excluded from inference, and the artifacts traced only to it disappear from the preview.
- **When** I re-enable it, **then** the app returns to the state it had with the story enabled (same model facts; the fixture engine makes this exact).

### US-A6 Delete a story
*As a Storyteller, I want to delete a story, so that I can drop an idea.*
Tags: `browser` · M1 · FR-1 · UC-03

- **When** I delete a story, **then** I can undo it until my next action, and the deletion is recorded in history.
- **Given** a component is traced to stories 1 and 3, **when** I delete story 3, **then** the component stays.

### US-A7 See what a story produced
*As a Storyteller, I want each story to show the screens, components and entities it produced, so that I know what my words did.*
Tags: `browser`, `unit` · M1 · FR-3 · UC-04

- **Given** story 1 produced `screen-login` and `cmp-login-form`, **then** story 1's trace lists both.
- **When** I click a trace item, **then** the preview navigates to it and highlights it.
- **Given** a story that produced nothing new, **then** its trace says which existing artifacts already satisfy it.

---

## Epic B: Inference and the App Model

### US-B1 One App Model is the source of truth
*As a Developer, I want a single App Model JSON that everything derives from, so that I can trust and inspect the app's definition.*
Tags: `unit`, `contract` · M0 · FR-4

- **Given** any version, **then** its App Model validates against the published schema (`app-model.schema.json`, versioned).
- **Given** a Claude response that does not validate, **then** it is rejected, the previous version stays current, and the story shows a failure (US-J1).
- **Given** the App Model, **then** every screen, component, entity and integration has a stable id and a non-empty list of source story ids.

### US-B2 Infer the app from the full story set
*As a Storyteller, I want each change to re-read every story, so that the app reflects my whole idea, not just the last line.*
Tags: `eval`, `browser` · M1 · FR-4, FR-5 · UC-02

Eval cases (live engine, scored on model facts, not wording):

| Scenario | After story | The App Model must contain |
| --- | --- | --- |
| Joe and Sally | 1 | platform includes `mobile`; persona Joe; an auth requirement; a login screen |
| Joe and Sally | 2 | platforms include `mobile` and `desktop` (responsive web); a desktop layout for home |
| Joe and Sally | 3 | persona Sally; entity Store; a store-context screen; at least one open assumption about the domain |
| Susie | 1 | weather domain; mobile-first; a home screen with forecast placeholder |
| Susie | 2 | integrations geolocation and weather API; entities Location and Forecast |
| Susie | 3 | place search; saved locations; multi-day forecast; local forecast still the first card |
| Susie | 4 | pivot flag set; app kind `game`; platform target iOS native distribution; App Store checklist present |

Browser checks (fixture engine): the preview after each step shows the screens and frames named in the PRD worked-example tables.

### US-B3 Every guess is an explicit assumption
*As a Storyteller, I want guesses shown with a confidence level, so that I can tell what I said from what Claude assumed.*
Tags: `contract`, `browser` · M3 · FR-6 · UC-05

- **Given** any model fact not stated by a story, **then** it is recorded as an assumption with confidence (0 to 1) and source story ids.
- **Given** Joe and Sally story 3, **then** the assumptions panel shows a domain guess (lists, inventory or similar) with confirm, reject and pin actions.

### US-B4 Confirm, reject or edit an assumption
*As a Storyteller, I want to answer an assumption, so that the app stops guessing about it.*
Tags: `browser`, `eval` · M3 · FR-6 · UC-05

- **When** I reject an assumption, **then** re-inference runs and the rejected value does not reappear in that or any later version unless I un-reject it (eval checks the live engine honours rejections across 3 more stories).
- **When** I confirm an assumption, **then** it moves to confirmed facts and is no longer shown as a guess.
- **When** I replace it with my own text, **then** my text is treated as a pinned fact.

### US-B5 Pin a decision
*As a Storyteller, I want to pin a decision, so that later stories cannot overturn it.*
Tags: `unit`, `browser`, `eval` · M3 · FR-6, FR-7 · UC-06

- **Given** I pin "keep it a web app", **when** I add "I want it on the App Store", **then** the platform stays web, and the change card says the story was partly overridden by a pin with an unpin action.
- **When** I unpin, **then** re-inference runs and the overridden part of the later story now applies.
- Pins are listed in one place and each can be removed.

### US-B6 Newest story wins a contradiction
*As a Storyteller, I want a later story to win over an earlier one, so that I can change my mind by writing.*
Tags: `unit`, `eval`, `browser` · M1 · FR-7 · UC-07

- **Given** "Joe logs into an app" then "Nobody needs to log in", **then** the login screen is removed and the change card names both stories and which won.
- **Then** the earlier story is marked as superseded in the list.
- **Given** the earlier decision is pinned, **then** the pin wins instead (see US-B5).

### US-B7 Pivots are detected and labelled
*As a Storyteller, I want an identity change to be called out, so that a big jump feels deliberate, not random.*
Tags: `eval`, `browser` · M4 · UC-08

- **Given** Susie story 4, **then** the version is flagged as a pivot, and the change card shows a before and after summary and a one-click rollback to the pre-pivot version.
- **Given** a non-pivot story (Susie story 3), **then** no pivot flag is set (eval measures false-pivot rate).

---

## Epic C: Generation

### US-C1 Generate vanilla web components
*As a Developer, I want plain HTML5 and custom elements with Shadow DOM, so that the output runs anywhere with no build step.*
Tags: `unit`, `browser` · M0 · FR-8

- **Given** any generated app, **then** it contains `index.html`, one ES module per custom element, and `store.js`, and imports no framework.
- **Given** the generated folder served by any static server, **when** `index.html` is opened in Chromium, Firefox and WebKit, **then** it renders with no console errors.
- **Then** every custom element name contains a hyphen, is registered once, and has a root `data-testid`.

### US-C2 Change only what the story needs
*As a Storyteller, I want untouched parts of the app to stay exactly the same, so that new stories do not break old ones.*
Tags: `unit`, `browser` · M1 · Vertical slices rule 2

- **Given** versions N and N+1 where the model diff touches only the Store entity and its screen, **then** every file not traced to those artifacts is byte-for-byte identical.
- **Given** a pivot version, **then** this rule is waived and the change card says so.

### US-C3 Each story is a working vertical slice
*As a Storyteller, I want each story to work end to end, so that I see a usable feature, not a mock-up.*
Tags: `browser` · M2 · Vertical slices

- **Given** Susie story 2 with stubbed geolocation and the mock weather adapter, **then** the forecast card shows the mocked local conditions, read through the data layer from a Forecast entity.

---

## Epic D: Preview

### US-D1 Render in inferred device frames
*As a Storyteller, I want the preview to show the devices my stories imply, so that I can see the app the way users will.*
Tags: `browser` · M1, M4 · FR-9

- **Given** platforms `[mobile]`, **then** one phone frame is shown.
- **Given** platforms `[mobile, desktop]`, **then** phone and desktop frames are shown side by side and each uses its own layout.
- **Given** a narrow tool window, **then** frames stack or can be switched with tabs (behaviour to confirm; see gaps).

### US-D2 Hot-swap without losing my place
*As a Storyteller, I want the preview to update in place, so that I do not lose what I was doing in the app.*
Tags: `browser` · M1 · FR-9 · UC-02

- **Given** I am on the home screen with text typed in a field, **when** a change touches only another screen, **then** the page does not reload, I stay on home, and my text is still there.
- **Given** a change that removes the screen I am on, **then** the preview moves to the app's start screen and says why.

### US-D3 Use the app for real inside the preview
*As a Storyteller, I want to click through and enter data, so that I can judge whether it works.*
Tags: `browser` · M2 · FR-9, FR-11 · UC-09

- **When** I add a saved place in Susie's app and a later story is processed, **then** the saved place is still there.
- **Given** an entity's shape changes, **then** existing preview data is migrated or reset with a visible notice (rule to confirm; see gaps).
- **Then** the generated app cannot read or write the tool's own storage or DOM (sandboxed iframe, separate origin).

### US-D4 Wireframe and styled modes
*As a Storyteller, I want to switch between wireframe and styled, so that I can focus on structure first.*
Tags: `browser` · M1 · FR-10 · UC-10

- **When** I toggle wireframe mode, **then** the preview turns grayscale with boxes and labels, no Claude call is made, and app state is kept.
- **Then** the generated acceptance tests pass in both modes.

---

## Epic E: Data and integrations

### US-E1 Data layer from entities
*As a Developer, I want a typed store generated from entities with IndexedDB persistence and seed data, so that slices have real data to work with.*
Tags: `unit`, `browser` · M2 · FR-11

- **Given** entities Location and Forecast, **then** `store.js` exposes create, read, update, delete and list for each, validates field types, and persists to IndexedDB.
- **Given** a fresh preview, **then** seed records exist so screens are not empty, and they are marked as seed data.

### US-E2 Mock services until I connect one
*As a Storyteller, I want external services mocked by default, so that the app works before I have any keys.*
Tags: `browser` · M2 · FR-12 · UC-11

- **Given** a weather integration with no key, **then** the forecast card shows mock data with a visible "sample data" marker.
- **When** I add a valid key, **then** live data replaces it and only the adapter module changes.
- **Given** an invalid key or a failing service, **then** the app falls back to mock data and shows the error in the integrations panel.

### US-E3 Keep secrets out of code
*As a Developer, I want API keys stored outside the generated code, so that an export never leaks them.*
Tags: `unit`, `browser` · M2 · Risks: insecure code

- **Given** a stored key, **then** no generated file and no exported file contains it (scan the output for the key string).

---

## Epic F: Change cards

### US-F1 Explain every change
*As a Storyteller, I want a change card after every change, so that I can follow why the app moved.*
Tags: `browser` · M1 · Core loop step 5 · UC-04

- **Given** Joe and Sally story 3, **then** the change card lists the added Store entity and store screen, each naming story 3 as the cause.
- **When** I click an item, **then** the preview and the story list highlight it.
- **Then** I can expand the card to see the model diff and the code diff.

---

## Epic G: History

### US-G1 Every change is a version
*As a Storyteller, I want a version per change, so that nothing I try is lost.*
Tags: `unit`, `browser` · M3 · FR-13 · UC-12

- **Given** 4 story changes, **then** history lists 4 versions after the initial empty one, each labelled with its cause and time.

### US-G2 Diff any two versions
Tags: `unit`, `browser` · M3 · FR-13

- **When** I select versions 2 and 4, **then** I see the model diff, the code diff, and the two previews side by side.

### US-G3 Roll back
*As a Storyteller, I want to roll back to any version, so that I can undo a direction I did not like.*
Tags: `unit`, `browser` · M3 · FR-13

- **When** I roll back to version 2, **then** stories, App Model, pins, assumption answers and code all match version 2 exactly, and the rollback is itself recorded as a new version.
- **Then** no Claude call is made for a rollback.

### US-G4 Come back later
*As a Storyteller, I want my projects saved, so that I can close the tool and continue tomorrow.*
Tags: `browser` · M1 · UC-16

- **Given** a project with 3 stories, **when** I reload the tool, **then** the project, its history and the preview data are restored.

---

## Epic H: Acceptance tests for the generated app

### US-H1 Each story gets a generated test
*As a Storyteller, I want every story checked automatically, so that I know later stories have not broken earlier ones.*
Tags: `generated`, `browser` · M3 · Vertical slices rule 5 · UC-13

- **Given** a story is processed, **then** a Playwright-style test file for it exists, selects elements by `data-testid` only, and passes against the new version.
- **Given** a new version is about to become current, **then** all story tests run; a version whose own story's test fails is not promoted (US-J1).
- **Given** a later story breaks an earlier story's test, **then** the earlier story shows a failing badge and the change card names the later story.

---

## Epic I: Platforms and export

### US-I1 Installable PWA when implied
Tags: `browser` · M4 · Platform inference

- **Given** a story mentioning offline use or the home screen, **then** the output includes a valid `manifest.json` and a service worker, and the app loads offline in the preview.

### US-I2 Game scaffold when implied
Tags: `browser` · M4 · Platform inference

- **Given** Susie story 4, **then** the app contains a canvas-based game web component with a running game loop (frame counter advances) inside an HIG-styled shell.

### US-I3 App Store readiness
*As a Storyteller, I want to know what stands between my app and the App Store, so that I am not surprised by review.*
Tags: `browser`, `unit` · M4 · Platform inference · UC-14

- **Given** a story mentioning the App Store, **then** the export includes a Capacitor project and the tool shows a checklist with privacy manifest, permission strings, Sign in with Apple when other social logins exist, in-app purchase rules, minimum functionality, icons and launch screens.
- **Then** each item is marked done, generated, or needs you.

### US-I4 Export to a folder
*As a Developer, I want to download the project, so that I can keep working on it myself.*
Tags: `browser`, `unit` · M5 · FR-14 · UC-15

- **When** I export, **then** the zip contains the app, the stories, the App Model, the generated tests and a README, and no secrets.
- **Given** the unzipped folder served statically, **then** the app runs and the generated tests pass.

### US-I5 Push to Git
Tags: `browser` · M5 · FR-14 · UC-15

- **When** I push to a repository I have connected, **then** a commit is created with the same contents as the folder export, and its message lists the stories since the previous push.

---

## Epic J: Failure and performance

### US-J1 Failures never break the current app
*As a Storyteller, I want a failed change to leave my app as it was, so that I can retry without losing anything.*
Tags: `contract`, `browser` · M1 · UC-17

- **Given** the engine times out, returns invalid JSON, or produces code that throws on load, **then** the previous version stays current, the story shows a failed badge with a short reason, and retry is offered.
- The fixture engine includes failure fixtures for each of these cases.

### US-J2 Fast enough to feel live
*As a Storyteller, I want to see a change soon after I submit, so that the loop stays fun.*
Tags: `eval` · M1 · Risks: latency

- **Given** the live engine and the worked examples, **then** the time from submit to first visible preview change is recorded per story; the target is set in the PRD (see gaps).
- **Then** a progress indicator appears within 200 ms of submit.

---

## Reserved `data-testid` names for the tool UI

| Area | testid |
| --- | --- |
| Story input and submit | `story-input`, `story-submit` |
| Story list and items | `story-list`, `story-item-{n}`, `story-edit-{n}`, `story-toggle-{n}`, `story-delete-{n}`, `story-trace-{n}`, `story-status-{n}` |
| Preview | `preview`, `device-frame-{mobile\|tablet\|desktop}`, `preview-mode-toggle` |
| Change card | `change-card`, `change-item-{artifactId}`, `change-card-pivot`, `change-card-expand` |
| Assumptions and pins | `assumptions-panel`, `assumption-{id}`, `assumption-confirm-{id}`, `assumption-reject-{id}`, `assumption-pin-{id}`, `pins-list`, `pin-{id}` |
| History | `history-panel`, `version-{n}`, `version-diff`, `version-rollback-{n}` |
| Integrations | `integrations-panel`, `integration-{id}`, `integration-key-{id}` |
| Tests | `story-test-badge-{n}` |
| Export | `export-folder`, `export-git` |

## Coverage map

| PRD item | Stories |
| --- | --- |
| FR-1 | A1, A3, A4, A5, A6 |
| FR-2 | A2 |
| FR-3 | A7 |
| FR-4 | B1, B2 |
| FR-5 | B2 |
| FR-6 | B3, B4, B5 |
| FR-7 | A4, B5, B6 |
| FR-8 | C1 |
| FR-9 | D1, D2, D3 |
| FR-10 | D4 |
| FR-11 | E1, D3 |
| FR-12 | E2 |
| FR-13 | G1, G2, G3 |
| FR-14 | I4, I5 |
| Worked example: Joe and Sally | B2 eval rows 1 to 3, F1 |
| Worked example: Susie | B2 eval rows 4 to 7, C3, E2, B7, I2, I3 |
| Change cards (core loop) | F1 |
| Per-story acceptance tests | H1 |
| Platform inference | D1, I1, I2, I3 |
| Not in PRD yet | A1 limits, D3 migration, G4 persistence, J1 failures, J2 latency target (see [prd-gaps.md](prd-gaps.md)) |
