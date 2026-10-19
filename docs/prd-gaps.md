# Story to Code: PRD gaps found while writing use cases and stories

Draft, 2026-10-19. These are things [PRD.md](PRD.md) does not yet decide that the [use cases](use-cases.md) and [user stories](user-stories.md) had to assume. Each gap lists the default the stories currently assume, so the PRD owner can accept it or replace it. Gaps are grouped by how much they block testing.

## Blocks testability

1. **No deterministic engine for tests.** The PRD makes Claude the core but says nothing about how the tool is tested without calling Claude. *Assumed:* inference and generation sit behind one engine interface, and tests use a fixture engine that replays recorded responses keyed by the story set; a separate eval suite runs the live engine nightly and reports a pass rate. (US-B2, US-J1, "How these become tests")
2. **Inference quality has no measure.** "Reliably produces a coherent app" (goal) has no pass bar. *Assumed:* the worked examples become eval cases scored on App Model facts (entities, screens, platforms, pivot flag), with a target pass rate the PRD should set. (US-B2, US-B7)
3. **The App Model has no published schema or ids.** FR-4 names the model but not its shape, versioning, or stable ids. Tests, traces, diffs and generated `data-testid`s all need stable ids. *Assumed:* a versioned JSON Schema, a stable id on every screen, component, entity and integration, and a list of source story ids on each. (US-B1)
4. **No selector contract for generated apps.** Generated acceptance tests break if they select by text or style. *Assumed:* every generated element root gets a `data-testid` derived from its model id. (US-C1, US-H1)
5. **Generated test failures have no consequence.** The PRD says each slice ships with a check but not what happens when it fails. *Assumed:* a version whose own story's test fails is not promoted; a later story breaking an earlier test flags the earlier story. (US-H1, US-J1)

## Missing behaviour

6. **Failure handling.** No requirement covers Claude timeouts, invalid structured output, or generated code that throws. *Assumed:* the last good version stays current, the story is marked failed with a reason, retry is offered. (UC-17, US-J1)
7. **Concurrent edits.** What happens when a story is added while the previous change is still processing is undefined. *Recommend deciding:* queue, or cancel and restart with the full set. (UC-02 alt 1a)
8. **Preview data across model changes.** When an entity's shape changes, data the person entered in the preview is neither migrated nor reset by any rule. *Assumed:* migrate when fields are only added; otherwise reset with a visible notice. (US-D3)
9. **Project persistence.** FR-13 versions everything, but nowhere says projects are saved and reopened, or where they live. *Assumed:* projects persist locally and restore fully on reload. (UC-16, US-G4) Ties to the open question on where the tool runs.
10. **Delete and undo.** Deleting a story can remove screens; there is no confirmation or undo rule. *Assumed:* undo until the next action, and deletion recorded in history. (US-A6)
11. **Empty and no-op stories.** No behaviour for a story that is off-topic ("hello") or adds nothing new. *Assumed:* off-topic yields the generic shell plus a low-confidence assumption; a no-op says which artifacts already satisfy it. (UC-01 alt 2b, UC-02 alt 3a)
12. **Rollback semantics.** FR-13 says roll back but not whether pins and assumption answers roll back too, or whether rollback destroys later versions. *Assumed:* everything rolls back together, and the rollback is a new version so nothing is lost. (US-G3)
13. **Superseded stories.** FR-7 resolves contradictions but does not say how the losing story is shown. *Assumed:* marked as superseded in the list. (US-B6)
14. **Pivot aftermath.** After Susie's game pivot, it is unclear whether location, search and saved places survive. *Recommend deciding:* earlier stories that no longer apply are flagged for the person rather than silently dropped. (UC-08)
15. **Reorder means newest.** FR-7 says newest wins; it is unstated whether "newest" is list order or creation time. *Assumed:* list order, so reordering can change the app. (US-A4)

## Missing numbers

16. **Latency target.** The PRD says "within seconds" and lists latency as a risk, but sets no target. *Recommend:* a target for submit-to-first-preview-change, and a progress indicator within 200 ms. (US-J2)
17. **Limits.** No maximum story length, story count per project, or token budget per change. (US-A1)
18. **Browser support.** "Runnable HTML5" without a target browser list. *Assumed:* current Chromium, Firefox and WebKit, which is also the Playwright matrix. (US-C1)

## Missing non-functional requirements

19. **Accessibility.** Neither the tool nor the generated apps have accessibility requirements (keyboard reordering, focus management on hot-swap, generated components with labels and roles). *Recommend:* WCAG 2.2 AA for the tool, and an automated axe check in each generated story test.
20. **Preview sandbox isolation.** "Sandboxed iframe" is named, but not what it must prevent. *Assumed:* the generated app cannot reach the tool's storage, DOM or API keys (separate origin). (US-D3)
21. **Secret storage.** FR-12 asks for keys but not where they are kept. *Assumed:* outside generated code and excluded from exports, checked by scanning output. (US-E3)
22. **Narrow screens.** Multiple device frames side by side have no layout rule for small windows. (US-D1)
23. **Privacy of stories.** Stories are sent to Claude; there is no statement on retention or what the person is told.
