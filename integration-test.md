# Integration Test Rules & Standards

## 1. Strictly Forbidden (MUST NOT DO)

- **NO Raw Assertions or Finders in Tests:** Test scripts (`*_test.dart`) must NEVER call `expect(...)`, `tester.tap(...)`, `find.byKey(...)`, or `find.byType(...)`.
- **NO Presentation Widget Imports in Tests:** Test scripts must NEVER import UI widgets (e.g., `package:ezrxmobile/presentation/.../order_history_item_tile.dart`) to inspect or locate them. All widget finders belong exclusively in Robot classes.
- **NO `if` Condition Bypasses:** NEVER wrap `expect()`, finders, or assertions inside `if` checks in tests, robots, or mixins (e.g., `if (finder.evaluate().isNotEmpty) expect(...)`). Every assertion must be a hard fail.
- **NO Silent Early Exits (`return;`):** NEVER write `if (items.isEmpty) { ... return; }` to exit early when data is missing. Vanishing assertions give false-positive passes; throw `TestFailure('Missing prerequisite data: ...')` instead.
- **NO Generic-Only Assertions on Search:** When searching for an item, NEVER verify only generic list presence (`verifyMaterial()`). MUST verify the exact target item (`verifyMaterialVisible(targetCode)`) to prevent false positives.
- **NO Blind Taps on Off-Screen Widgets:** NEVER call `tester.tap(...)` directly on elements that might be scrolled off-screen (causes `Bad state: No element`). Always call `tester.scrollEnsureFinderVisible(finder)` or `tester.dragUntilVisible(...)` first.
- **NO Blind `pumpAndSettle` on Infinite Loaders:** NEVER call `tester.pumpAndSettle()` while looping shimmers or spinners are active (hangs until Patrol test runner times out). Use bounded polling loops with `tester.pump(duration)` until loading disappears.
- **NO Helper Method Proliferation:** NEVER create one-off wrapper functions just to pass different arguments (e.g., DO NOT create `tapSortAZToggle()`; use `tapSortToggle(Sort.az.title)`).
- **NO Duplicate Robot Logic:** NEVER write new methods or navigation helpers if they already exist in parent robots (`CommonRobot`), mixins (`MarketPlaceRobotMixin`), or shared helpers (`orderingIntegrationTestCommon`).
- **NO Dead Code or Hardcoded Data in Robots:** NEVER add unused fields, getters, or hardcoded strings to Robot classes. Robots only handle UI interactions and assertions.
- **NO Unofficial Steps:** NEVER invent steps, skip validations, or deviate from the Zephyr specification.
- **NO Circuitous Navigation:** NEVER navigate through extra screens when the action is available on the current screen (e.g., tap the search field on the Product tab directly; do not return to Home).
- **NO Hardcoded Localized Text:** NEVER match UI text with hardcoded English strings via `find.text('...')`. Use `WidgetKeys` or `'...'.tr()` so tests pass across all markets (VN, TH, TW, etc.).
- **NO Arbitrary Sleeps:** NEVER use `Future.delayed()` or `sleep()`. Use `tester.pumpAndSettle()` or `tester.pumpUntilVisible()`.
- **NO Smart Quotes:** NEVER write Unicode curly quotes (`‘`, `’`, `“`, `”`) in `.dart` or `.json` files. Use strictly ASCII (`'`, `"`) (blocked by pre-push hook).
- **NO Shared State Leaks:** NEVER leave account mutations (e.g., favorited items, cart items) uncleaned across test runs. Always use `clearCart: true` or post-test teardown.
- **NO Illegal Characters in Descriptions:** NEVER use `/`, `&`, `[`, or `]` in `testCaseWrapper` titles (crashes Android test runner; use `and` instead).
- **NO Backend Config Mutations:** Mobile integration tests must NEVER attempt to mutate backend Sales Org configs (e.g., toggling MP settings in back-office portals).

---

## 2. Mandatory Requirements (MUST DO)

- **100% Robot Encapsulation:** Encapsulate all UI taps, scrolls, visibility checks, and assertions inside Robot classes or extensions.
- **Reuse Existing Code First:** Always search the existing Robot, parent `CommonRobot`, and shared Mixins before adding a new method.
- **Parameterize Over Duplication:** Use parameters or optional arguments (e.g., `verifyMPSubTotal([String? price])`, `verifySortToggle({required String sort, bool isVisible = true, bool isSelected = false})`).
- **Explicit `TestFailure` on Missing Preconditions:** If test data or pre-existing state is insufficient (e.g., listing has fewer items than required page size), throw `throw TestFailure('Reason')` rather than silently passing.
- **Strict Zephyr Exactness:** Execute every step and expected result precisely as documented in Zephyr. Report anomalies immediately.
- **Document Non-Applicable Steps:** If a Zephyr step is marked as "Web only" or requires back-office config mutations, add an explicit comment (e.g., `// Step X: Skipped - Web only (...)`) instead of omitting it silently.
- **Marketplace Context:** Always pass `customerCodeType: .marketPlace` when testing Marketplace features. Never use `.defaultCode`.
- **Complete Role Coverage:** Include all valid personas in `roles: [...]` (e.g., `.salesRep` can place orders and browse catalog; do not drop `.salesRep` without explicit business reason).
- **Explicit Market Scoping:** If a feature is restricted to specific countries (e.g., MP batch/expiry for MY, PH, VN), scope it explicitly in `testCaseWrapper(markets: const [...])`.
- **RichText Flag for Formatted Text:** When matching prices, badges, or formatted labels in `Text.rich` / `RichText`, always pass `findRichText: true` (e.g. `find.text(price, findRichText: true)`).
- **Keyboard Dismissal Before Footer Taps:** Always close the virtual keyboard (`unfocus` or keyboard action) before tapping sticky buttons or bottom sheet footers obscured by the keyboard.
- **Standard Suite Structure:** Implement `_liveTestData`, `_placeholderTestData`, and `static void useTestData(...)` in every test class for `runner_test.dart` compatibility.
- **Data-Driven Fixtures:** Load test inputs and expected values dynamically via `JsonSessionTestData` / `DataLoader`.
- **Central Runner Only:** Register test suites in `integration_test/tests/runner_test.dart` and execute via Patrol.
- **Commit & PR Standard:** Commit titles must follow `test(EZRX-XXXXX): [Mobile] Integration Tests - <Domain> - <Feature>` and pass `fvm flutter analyze integration_test` with 0 errors and 0 warnings before opening a PR.
