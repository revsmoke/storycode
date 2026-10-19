# Story to Code: User stories and acceptance criteria

Draft, 2026-10-19. Companion to [PRD.md](PRD.md) and [use-cases.md](use-cases.md). Aligned with the PRD as revised by the gap review; where a criterion goes beyond what the PRD states, it is marked *(assumed)*.

These stories describe Story to Code itself. Each one has acceptance criteria written as Given / When / Then so they can become automated tests later without rewording. Every story is tagged with the test level it belongs to, the milestone it lands in, and the PRD requirement and use case it traces to.

## How these become tests

Story to Code has a non-deterministic core (Claude) wrapped in deterministic plumbing. The test plan splits along that line, following the PRD's "Testing the tool" and "Acceptance tests" sections.

| Level | Tag | What it checks | Runs against | Where |
| --- | --- | --- | --- | --- |
| Unit | `unit` | Pure logic: App Model schema validation, model diffing, version store, trace bookkeeping, pin and contradiction rules | Plain JS modules | Node test runner, every commit |
| Contract | `contract` | Claude responses parse into a valid App Model or code patch; malformed output is rejected | Recorded fixtures plus a schema | Every commit |
| Browser | `browser` | The tool's UI and the generated app behave as described | Playwright with the **fixture engine** (below) | Every commit |
| Eval | `eval` | Inference quality: does story X actually produce the expected model? | The live Claude API, scored against expected model facts | Nightly; reports pass rate and median latency. v1 bar: every step of both worked examples passes in at least 9 of 10 runs |
| Generated | `generated` | The per-story acceptance tests Story to Code writes for the user's app | The generated app, in the preview's built-in runner | Inside the product (UC-13), and as Playwright specs in exports |

**The fixture engine.** Browser tests must not call Claude. Inference and generation sit behind one engine interface, and tests swap in a fixture engine that replays recorded responses keyed by a hash of the request (story set, model, pins, dial). The two PRD worked examples (Joe and Sally, Susie) are the first fixture sets. Fixtures are re-recorded from the live engine when the prompt, schema or conventions change, and the fixture diff is reviewed like code. Eval tests use the same story sets with the live engine, so a fixture and an eval case are the same scenario at two levels.

**Selectors.** Browser tests address UI by `data-testid`, never by text or CSS. The tool reserves the names in the table at the end of this file *(assumed; the PRD's selector contract covers generated apps only)*. Generated apps follow the PRD's selector contract: every element root and interactive control carries a `data-testid` derived from its model id (`screen-home`, `weather-card`, `place-search-input`), so generated tests stay stable across restyles and wireframe mode.

**Time and network.** Browser tests stub geolocation, freeze the clock, and route external APIs to the mock adapters (FR-12), so the Susie scenario is reproducible.

## Story format

Each story uses the PRD's optional structure (FR-2): *As a … I want … so that …*, followed by acceptance criteria. IDs are stable; add new stories at the end of their epic rather than renumbering. In criteria, `s1`, `s2`… are the stable story ids the PRD assigns at creation.

---

## Epic A: Stories (writing and managing them)

### US-A1 Write a story in free text
*As a Storyteller, I want to type a story in plain language, so that I can describe my idea without learning a format.*
Tags: `browser` · M0 · FR-1 · UC-01

- **Given** an empty project, **when** I type "Joe logs into an app." and submit, **then** the story appears first in the list with a stable id and processing starts.
- **Given** the story input is empty or whitespace, **then** submit is disabled.
- **Given** a story of exactly 2,000 characters, **when** I submit it, **then** it is accepted in full; at 2,001 characters submit is blocked and the limit is shown.
- **Given** a project with 200 stories, **when** I try to add another, **then** the tool asks me to split the project.
- **Given** I submit with the keyboard (Enter, or Cmd/Ctrl+Enter in a multi-line input), **then** it behaves the same as the submit button.

### US-A2 Optionally use structured form
*As a Storyteller who knows the format, I want to write "As a…, I want…, so that…" and add acceptance criteria, so that the engine gets more precise input.*
Tags: `unit`, `browser` · M1 · FR-2

- **Given** a story in structured form, **when** it is parsed, **then** persona, want and reason are stored as separate fields and shown as such.
- **Given** a story with acceptance criteria, **then** that story's generated acceptance test is written from them (US-H1).
- **Given** a free-text story, **then** no structure is required and no warning is shown.

### US-A3 Edit a story
*As a Storyteller, I want to edit a story I already wrote, so that I can refine the idea.*
Tags: `browser` · M1 · FR-1 · UC-03

- **Given** story `s2` exists, **when** I edit its text and save, **then** re-inference runs, `s2` keeps its id, and the new version's change card names `s2` as the cause.
- **Given** I start editing and cancel, **then** nothing changes and no version is recorded.
- **Given** a change is in flight, **when** I edit a story, **then** the in-flight change is cancelled and restarts after a short debounce, and the preview keeps the last good version until the new one is ready (FR-17).

### US-A4 Reorder stories
*As a Storyteller, I want to drag stories into a different order, so that I can control which one wins a conflict.*
Tags: `browser`, `unit` · M1 · FR-1, FR-7 · UC-03

- **Given** stories `s1` to `s3`, **when** I drag `s3` above `s1`, **then** the list order and the stored order both change, every id is unchanged, and re-inference runs.
- **Given** two conflicting stories with no pin, **when** I swap their order, **then** the winner swaps too, since the story lower in the list wins (unit test on the precedence rule).
- Reordering is also possible with the keyboard (WCAG 2.2 AA).

### US-A5 Disable and re-enable a story
*As a Storyteller, I want to switch a story off without deleting it, so that I can try the app without it.*
Tags: `browser` · M1 · FR-1 · UC-03

- **Given** story `s3` is enabled, **when** I disable it, **then** it is visibly dimmed, excluded from inference, and the artifacts traced only to it disappear from the preview.
- **When** I re-enable it, **then** the app returns to the state it had with the story enabled (same model facts; the fixture engine makes this exact).

### US-A6 Delete a story
*As a Storyteller, I want to delete a story, so that I can drop an idea.*
Tags: `browser` · M1 · FR-1 · UC-03

- **When** I delete a story, **then** I can undo it until the next change, and the deletion is recorded in history.
- **Given** a component is traced to `s1` and `s3`, **when** I delete `s3`, **then** the component stays and its trace becomes `[s1]`.
- **Given** an artifact traced only to `s3`, **when** I delete `s3`, **then** the artifact is removed (empty `stories` list).

### US-A7 See what a story produced
*As a Storyteller, I want each story to show the screens, components and entities it produced, so that I know what my words did.*
Tags: `browser`, `unit` · M1 · FR-3 · UC-04

- **Given** `s1` produced the login screen and `<login-form>`, **then** `s1`'s trace lists both, and each artifact's `stories` in the App Model contains `s1`.
- **When** I click a trace item, **then** the preview navigates to it and highlights it *(assumed)*.
- **Given** a story the app already satisfies, **then** a no-op change card names the artifacts that cover it.
- **Given** a story too vague to change the app ("Joe is happy"), **then** no slice is produced and the change card suggests what the story could add.
- **Given** a persona-only story ("Joe is a store manager"), **then** a fact is added to persona Joe and no screen is produced.
- **Given** any generated file, **then** it starts with a comment naming the story ids it serves.

---

## Epic B: Inference and the App Model

### US-B1 One App Model is the source of truth
*As a Developer, I want a single App Model JSON that everything derives from, so that I can trust and inspect the app's definition.*
Tags: `unit`, `contract` · M0 · FR-4

- **Given** any version, **then** its App Model validates against the schema for its `schemaVersion`.
- **Given** inference returns a JSON Patch that does not validate, **then** the error is sent back once; on a second failure the change is abandoned, the last good model stays, and an error card is shown (US-J1).
- **Given** the App Model, **then** every screen, component, entity and integration has an id (or tag or name) and a non-empty `stories` list; an artifact whose list becomes empty is removed on the next change.
- **Given** a saved project from an older `schemaVersion`, **then** it opens through a migration.
- **Given** any inference request, **then** it contains no generated code, only the model, stories, pins and rejected assumptions.

### US-B2 Infer the app from the full story set
*As a Storyteller, I want each change to re-read every story, so that the app reflects my whole idea, not just the last line.*
Tags: `eval`, `browser` · M1 · FR-4, FR-5 · UC-02

Eval cases (live engine, scored on model facts, not wording):

| Scenario | After story | The App Model must contain |
| --- | --- | --- |
| Joe and Sally | 1 | platform includes `mobile`; persona Joe; an auth requirement; a login screen |
| Joe and Sally | 2 | platforms include `mobile` and `desktop` (responsive web); a desktop layout for home |
| Joe and Sally | 3 | persona Sally; entity Store; a store-context screen; at least one open assumption about the domain; if shared data between Joe and Sally is implied, an assumption naming the missing backend |
| Susie | 1 | weather domain; mobile-first; a home screen with forecast placeholder |
| Susie | 2 | integrations geolocation and weather API; entities Location and Forecast |
| Susie | 3 | place search; saved locations; multi-day forecast; local forecast still the first card |
| Susie | 4 | pivot flag set; app kind `game`; platform target iOS native distribution; App Store checklist present |

Browser checks (fixture engine): the preview after each step shows the screens and frames named in the PRD worked-example tables.

Determinism (eval): the same story set, pins and dial produce the same screens, components and entities (names and structure) across runs.

### US-B3 Every guess is an explicit assumption
*As a Storyteller, I want guesses shown with a confidence level, so that I can tell what I said from what Claude assumed.*
Tags: `contract`, `browser` · M3 · FR-6 · UC-05

- **Given** any model fact not stated by a story, **then** it is recorded as an assumption with confidence `low`, `medium` or `high`, its source story ids, and the model `path` it shaped.
- **Given** Joe and Sally story 3, **then** the Assumptions pane shows a domain guess (lists, inventory or similar) with confirm, reject and pin actions.

### US-B4 Confirm, reject or edit an assumption
*As a Storyteller, I want to answer an assumption, so that the app stops guessing about it.*
Tags: `browser`, `eval` · M3 · FR-6 · UC-05

- **When** I reject an assumption, **then** it is added to the model's `rejected` list, re-inference runs with it as a constraint, and the same guess does not reappear in later versions (eval checks the live engine honours rejections across 3 more stories).
- **When** I confirm an assumption, **then** its confidence rises and it leaves the pane.
- **When** I pin an assumption, **then** a pin is added at the assumption's `path`.

### US-B5 Pin a decision
*As a Storyteller, I want to pin a decision, so that later stories cannot overturn it.*
Tags: `unit`, `browser`, `eval` · M3 · FR-6, FR-7 · UC-06

- **Given** a pin on `/platform/distribution` = `web`, **when** I add "I want it on the App Store", **then** the validator rejects any patch that changes the pinned path, the platform stays web, and the change card names the pin that blocked it with an unpin-and-retry action (unit test on the validator; eval on the live engine).
- **When** I unpin and retry, **then** the blocked part of the later story now applies.
- Pins are listed in the Assumptions pane and each can be removed.
- Pins hold under both dial settings (US-B8).

### US-B6 The story lower in the list wins a conflict
*As a Storyteller, I want a later story to win over an earlier one, so that I can change my mind by writing.*
Tags: `unit`, `eval`, `browser` · M1 · FR-7 · UC-07

- **Given** "Joe logs into an app" then "Nobody needs to log in" appended below it, **then** the login screen is removed and the change card names both stories and which won.
- **Then** the earlier story is marked superseded, naming the story that overrode it.
- **When** I move the superseded story below the winner, **then** it wins again.
- **Given** the earlier decision is pinned, **then** the pin wins instead (see US-B5).

### US-B7 Pivots are detected and labelled
*As a Storyteller, I want an identity change to be called out, so that a big jump feels deliberate, not random.*
Tags: `eval`, `browser` · M4 · UC-08

- **Given** Susie story 4, **then** the version is flagged as a pivot, and the change card shows a pivot badge and a one-click rollback to the pre-pivot version.
- **Then** earlier stories the pivot no longer satisfies (location, search, saved places) are flagged on the card and in the list, not silently dropped.
- **Given** a non-pivot story (Susie story 3), **then** no pivot flag is set (eval measures false-pivot rate).

### US-B8 Steady or wild
*As a Storyteller, I want to choose how freely Claude reinterprets my app, so that I control how much it surprises me.*
Tags: `unit`, `eval`, `browser` · M1 · FR-19; Inference contract: the dial

- **Given** a new project, **then** the dial in Settings defaults to steady, and the first story runs as wild.
- **Given** steady, **when** a story could be met by an existing screen, component or entity, **then** inference reuses it rather than adding a new one (eval).
- **Given** wild, **then** inference may re-plan the whole model, and pins still hold.

---

## Epic C: Generation

### US-C1 Generate vanilla web components
*As a Developer, I want plain HTML5 and custom elements with Shadow DOM, so that the output runs anywhere with no build step.*
Tags: `unit`, `browser` · M0 · FR-8

- **Given** any generated app, **then** it follows the PRD layout (`index.html`, `app-shell.js`, `components/<tag>.js`, `store.js`, `adapters/<id>.js`, `theme.css`, `tests/<story-id>.test.js`, `app-model.json`, `stories.json`) and imports no framework.
- **Given** the generated folder served by any static server, **when** `index.html` is opened in current Chrome, Safari, Firefox and Edge (Playwright: Chromium, WebKit, Firefox), **then** it renders with no console errors.
- **Then** routing is hash-based and the app also works from `file://`.
- **Then** every custom element name contains a hyphen, is registered once, uses Shadow DOM, and carries a root `data-testid`; every interactive control carries one too.
- **Then** components contain no `fetch` calls (data only through `store`, services only through an adapter), and lint rejects inline event handlers, `eval` and key-shaped literals.

### US-C2 Change only what the story needs
*As a Storyteller, I want untouched parts of the app to stay exactly the same, so that new stories do not break old ones.*
Tags: `unit`, `browser` · M1 · Vertical slices rule 2

- **Given** versions N and N+1 where the model diff touches only the Store entity and its screen, **then** every component whose model entry hash is unchanged is byte-for-byte identical.
- **Given** a pivot version, **then** this rule is waived and the change card carries the pivot badge.

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
- **Given** a narrow tool window, **then** the frames become tabs (one visible, the others one click away) and the story list collapses to a drawer.

### US-D2 Hot-swap without losing my place
*As a Storyteller, I want the preview to update in place, so that I do not lose what I was doing in the app.*
Tags: `browser` · M1 · FR-9 · UC-02

- **Given** I am on the home screen with text typed in a field, **when** a change touches only another screen, **then** the page does not reload, changed modules are re-imported by hashed URL, I stay on home, my text is still there, and keyboard focus stays in place.
- **Given** a change that removes the screen I am on, **then** the preview moves to the app's start screen and says why *(assumed)*.

### US-D3 Use the app for real inside the preview
*As a Storyteller, I want to click through and enter data, so that I can judge whether it works.*
Tags: `browser` · M2 · FR-9, FR-11 · UC-09

- **When** I add a saved place in Susie's app and a later story is processed, **then** the saved place is still there.
- **Given** a saved place exists, **when** a change adds a field to Location, **then** the record keeps its data and the new field gets its default; **when** a field is renamed, **then** its value is mapped; **when** a field is removed, **then** its value is copied to the `_archive` store.
- **Then** the generated app cannot read or write the tool's DOM, storage or keys: the iframe is on a separate origin with `sandbox="allow-scripts allow-same-origin"`, and a CSP limits scripts to the preview origin and network calls to the tool's proxy.
- **Then** `allow="geolocation"` is present only when the model asks for location.
- **When** generated code throws at runtime, **then** the error is reported back to the tool and shown on the change card.

### US-D4 Wireframe and styled modes
*As a Storyteller, I want to switch between wireframe and styled, so that I can focus on structure first.*
Tags: `browser` · M1 · FR-10 · UC-10

- **When** I toggle wireframe mode, **then** the `theme.css` token set swaps to grayscale with boxes and labels, no Claude call is made, components are not regenerated, and app state is kept.
- **Then** the generated acceptance tests pass in both modes, and wireframe mode keeps readable contrast.

---

## Epic E: Data and integrations

### US-E1 Data layer from entities
*As a Developer, I want a typed store generated from entities with IndexedDB persistence and seed data, so that slices have real data to work with.*
Tags: `unit`, `browser` · M2 · FR-11; Data layer

- **Given** entities Location and Forecast, **then** `store.js` exposes `list`, `get`, `put`, `delete` and `query` for each, plus `store.subscribe(entity, callback)`, and persists to one IndexedDB database with one object store per entity, versioned by the App Model version.
- **When** a record changes, **then** subscribed components re-render.
- **Given** a fresh preview, **then** seed records exist so screens are not empty, and I can clear them from Settings.
- **Given** stories that imply data shared between people (Joe and Sally), **then** the app stays client-only and an assumption names the missing backend.

### US-E2 Mock services until I connect one
*As a Storyteller, I want external services mocked by default, so that the app works before I have any keys.*
Tags: `browser` · M2 · FR-12 · UC-11

- **Given** a weather integration with no key, **then** the adapter's `mock` flag is true and it returns fixtures matching the entity shape; the card shows a "sample data" marker *(assumed)*.
- **When** I add a valid key in Settings, **then** live data replaces it without regenerating any code; in the preview the adapter calls `/proxy/weather`, which adds the key.
- **Given** an invalid key or a failing service, **then** the app falls back to mock data and shows the error in Settings *(assumed)*.

### US-E3 Keep secrets out of code
*As a Developer, I want API keys stored outside the generated code, so that an export never leaks them.*
Tags: `unit`, `browser` · M2 · Risks: insecure code

- **Given** a stored key, **then** no generated file, App Model or export contains it (scan the output for the key string and for key-shaped literals).
- **Given** an export, **then** adapters read keys from `config.js`, and the export's `.gitignore` excludes it.

---

## Epic F: Change cards

### US-F1 Explain every change
*As a Storyteller, I want a change card after every change, so that I can follow why the app moved.*
Tags: `browser` · M1 · FR-16 · UC-04

- **Given** Joe and Sally story 3, **then** the change card has a title, names `s3` as the cause, summarises the model diff (Store entity and store screen added), and lists the files touched, new assumptions, inferred acceptance criteria and test results.
- **When** I click an item, **then** the preview and the story list highlight it *(assumed)*.
- **Then** the card has buttons to roll back and to open the diff.
- **Then** the Changes pane lists all cards newest first, with pivots badged.

---

## Epic G: History

### US-G1 Every change is a version
*As a Storyteller, I want a version per change, so that nothing I try is lost.*
Tags: `unit`, `browser` · M3 · FR-13 · UC-12

- **Given** 4 successful story changes, **then** the Versions pane lists 4 versions, each labelled with its change card title.
- **Given** a failed change, **then** no version is created.
- **Then** a version holds the story set, App Model, code snapshot, test results and change card; files unchanged between versions are stored once (content-addressed).

### US-G2 Diff any two versions
Tags: `unit`, `browser` · M3 · FR-13

- **When** I diff version 2 against version 4, **then** I see the structured model diff (added, removed, changed artifacts) and the code diff per file.
- **When** I click a version, **then** the preview shows it without making it current.

### US-G3 Roll back
*As a Storyteller, I want to roll back to any version, so that I can undo a direction I did not like.*
Tags: `unit`, `browser` · M3 · FR-13

- **When** I roll back to version 2, **then** the story list, App Model, pins, assumption answers and code all match version 2 exactly, and the rollback is recorded as a new version.
- **When** I roll back the rollback, **then** the pre-rollback state returns.
- **Then** no Claude call is made for a rollback.

### US-G4 Come back later
*As a Storyteller, I want my projects saved, so that I can close the tool and continue tomorrow.*
Tags: `browser` · M1 · UC-16

- **Given** a project with 3 stories, a pin and preview data, **when** I reload the tool, **then** stories, versions, pins, settings and the preview's IndexedDB data are restored exactly.

---

## Epic H: Acceptance tests for the generated app

### US-H1 Each story gets a generated test
*As a Storyteller, I want every story checked automatically, so that I know later stories have not broken earlier ones.*
Tags: `generated`, `browser` · M3 · Acceptance tests · UC-13

- **Given** story `s2` is processed, **then** `tests/s2.test.js` exists, selects elements by `data-testid` only, asserts on both the DOM and the store, and passes against the new version in the preview's built-in runner.
- **Given** a story without acceptance criteria, **then** Claude infers criteria and lists them on the change card, where I can correct them.
- **Then** each test runs an axe accessibility check on the screens it visits.
- **Given** a new version, **then** all story tests run; a version whose own story's test fails is not promoted, and the preview keeps the last good version until a repair passes.
- **Given** a later story breaks an earlier story's test, **then** the earlier story shows a regression badge and the change card offers a repair scoped to that slice.

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

- **Given** a story mentioning the App Store, **then** the export includes a `native/` Capacitor project that builds, and `native/APP_STORE_CHECKLIST.md` covering privacy manifest, permission strings, Sign in with Apple when other social logins exist, in-app purchase rules, minimum functionality, icons and launch screens.
- **Then** each item is marked done, generated, or needs you *(assumed)*.

### US-I4 Export to a folder
*As a Developer, I want to download the project, so that I can keep working on it myself.*
Tags: `browser`, `unit` · M5 · FR-14 · UC-15

- **When** I export, **then** the folder contains `app/` (with `stories.json`, `app-model.json` and the tests), a `README.md` on running it, supplying keys and running tests, and a `CHANGES.md` built from the change cards newest first; it contains no secrets.
- **Given** the export on a clean machine with only keys added, **then** the app runs from a static server and `npx playwright test` passes.
- **Then** the README says hand edits are not read back and names the eject point.

### US-I5 Push to Git
Tags: `browser` · M5 · FR-14 · UC-15

- **When** I export to Git, **then** there is one commit per version, each with its change card title as the message, so `git log` reads as the story history.
- **When** I push to a remote, **then** it uses my GitHub connection, and no Git token appears in the export.

---

## Epic J: Failure and performance

### US-J1 Failures never break the current app
*As a Storyteller, I want a failed change to leave my app as it was, so that I can retry without losing anything.*
Tags: `contract`, `browser` · M1 · FR-18 · UC-17

- **Given** Claude is unreachable, returns an invalid model twice, or generated code throws on load, **then** the preview never blanks, the last good version stays current, and an error card explains and offers retry.
- The fixture engine includes failure fixtures for each of these cases.

### US-J2 Fast enough to feel live
*As a Storyteller, I want to see a change soon after I submit, so that the loop stays fun.*
Tags: `eval` · M1 · Risks: latency

- **Then** a progress indicator appears within 200 ms of a story change (browser, fixture engine).
- **Given** the live engine and a story set under 20 stories, **then** the change card appears within a few seconds and the preview updates within 10 seconds; the nightly eval reports median and 90th percentile (eval).
- **Then** Settings show the project's token spend to date.

---

## Reserved `data-testid` names for the tool UI

| Area | testid |
| --- | --- |
| Story input and submit | `story-input`, `story-submit` |
| Story list and items | `story-list`, `story-item-{storyId}`, `story-edit-{storyId}`, `story-toggle-{storyId}`, `story-delete-{storyId}`, `story-trace-{storyId}`, `story-status-{storyId}`, `story-undo` |
| Preview | `preview`, `device-frame-{mobile\|tablet\|desktop}`, `device-frame-tab-{target}`, `preview-mode-toggle` |
| Change card and Changes pane | `change-card`, `change-item-{artifactId}`, `change-card-pivot`, `change-card-diff`, `change-card-rollback`, `change-card-repair`, `error-card`, `error-retry`, `changes-pane` |
| Assumptions pane and pins | `assumptions-pane`, `assumption-{id}`, `assumption-confirm-{id}`, `assumption-reject-{id}`, `assumption-pin-{id}`, `pins-list`, `pin-{index}`, `pin-remove-{index}` |
| Versions pane | `versions-pane`, `version-{n}`, `version-diff-{n}`, `version-rollback-{n}` |
| Settings pane | `settings-pane`, `settings-claude-key`, `integration-{id}`, `integration-key-{id}`, `dial`, `seed-clear`, `token-spend` |
| Tests | `story-test-badge-{storyId}` |
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
| FR-15 | B3, B4 |
| FR-16 | F1, B7 |
| FR-17 | A3 |
| FR-18 | J1, B1 |
| FR-19 | E2, E3, B8 |
| Stories and precedence | A1, A4, A6, A7, B6 |
| App Model schema and inference contract | B1, B5, B8 |
| Generated code conventions | C1, C2 |
| Data layer | E1, D3 |
| Preview sandbox and keys | D1, D2, D3, E2, E3 |
| Acceptance tests | H1 |
| Versions | G1, G2, G3, G4 |
| Export | I4, I5 |
| NFRs: latency, browsers, accessibility, determinism | J2, C1, A4, D2, H1, B2 |
| Worked example: Joe and Sally | B2 eval rows 1 to 3, F1 |
| Worked example: Susie | B2 eval rows 4 to 7, C3, E2, B7, I2, I3 |
| Platform inference | D1, I1, I2, I3 |

Criteria marked *(assumed)* are the only places these stories go beyond the PRD: highlighting from a trace or change item, the start-screen fallback when the current screen is removed, the sample-data marker and mock fallback for a bad key, checklist item states, and the tool's own `data-testid` names.
