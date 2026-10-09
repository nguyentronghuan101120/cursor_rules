---
description: Workflow for writing Flutter integration tests from Zephyr test cases.
alwaysApply: false
---

# Integration Test Workflow

## 1. Core Principles & Directives

1. **Ask ONE question at a time.** Wait for user response. NEVER assume or auto-generate.
   - **Exception:** When the user issues an explicit directive (e.g. "create X", "do A"), execute immediately. State any assumption in 1 line mid-flight.
2. **Strict Zephyr Fidelity (Absolute Requirement):**
   - **Carefully read all Steps, Test Data, and Expected Results in Zephyr.**
   - **Mandatory Exact Execution:** Strictly follow every step, test data specification, and expected result precisely as defined in Zephyr without omitting anything. No matter how unreasonable, illogical, or redundant it may seem, you MUST follow Zephyr exactly.
   - **Suspicious Test Steps:** If an anomaly, conflict, or outdated step is detected: **STOP AND REPORT IMMEDIATELY TO THE USER**. Never alter test steps, skip validations, or assume workarounds autonomously.
3. **Zero-Tolerance for `if` Bypass:**
   - In both test scripts and Robot classes / Mixins: **STRICTLY FORBIDDEN** to wrap `expect(...)`, finders, or actions inside `if` statements (prevents false positives where tests pass despite UI bugs).
   - Forbidden anti-patterns:
     - `if (finder.evaluate().isNotEmpty) expect(...)` -> FORBIDDEN! Use direct `expect(finder, findsOneWidget)` or `findsAtLeastNWidgets(1)`.
     - `if (line.contains(label)) expect(...)` -> FORBIDDEN! Skips validation when label is missing.
     - `if (stockInfo.batch.isNotEmpty) expect(...)` -> FORBIDDEN! Skips validation when data is empty.
     - `if (isVisible) expect(...)` -> FORBIDDEN! Silently skips assertions on failure.
   - Every verification MUST be a hard assertion (direct `expect(...)`). If UI fails to render or data is invalid, the test **MUST FAIL**.
   - When encountering legacy code with `if` conditions that skip assertions: create a new function or refactor to remove all `if` statements.

---

## 2. Architectural Invariants

All mobile integration tests follow a 3-layer architecture:
- **Layer 1: Robots** (`integration_test/robots/`): Page drivers handling UI interactions and assertions. Accessed via `locator<RobotName>()`.
- **Layer 2: Core & Data** (`integration_test/core/` & `integration_test/data/`): DI (`test_setup.dart`), context, loader (`data_loader.dart`), and fixtures (`data/users/{market}/{role}.json`).
- **Layer 3: Test Cases** (`integration_test/tests/`): Centralized tests grouped by domain (e.g. `tests/products/`, `tests/cart/`).

### Mandatory Invariants:
- **New Tests Placement**: All new test files MUST be placed under `integration_test/tests/` (NEVER in legacy per-market folders like `integration_test/ph/`).
- **Robot Pattern Enforced**: Test files (`*_test.dart`) must NEVER invoke `tester.tap(...)` or `find.byKey(...)` directly. All UI interactions belong in Robot classes.
- **WidgetKeys Mandatory**: Locate widgets using `WidgetKeys` (`lib/presentation/core/widget_keys.dart`). Avoid fragile `find.text()`.
- **Central Runner Only**: Tests MUST ONLY execute via `integration_test/tests/runner_test.dart` using `patrol test`. Never run individual test files directly.
- **Traceability & Description Format**: Every test MUST use `testCaseWrapper` with the Zephyr ID: `EZRX-TXXXX | Description`.
  - **No `/` or `&` characters**: NEVER use `/` or `&` in test descriptions (crashes Android test runner; use `and` instead). Avoid square brackets `[` `]`.
- **Marketplace Customer Code Requirement**:
  - For all Marketplace (MP) tests (e.g. MP search, MP filter, MP listing, MP logo, sold by, batch/expiry): **ALWAYS** pass `customerCodeType: .marketPlace` to `pumpAppOnHomePage(...)` or `pumpAndNavigateToProductListingPage(...)`.
  - **NEVER** use the default customer code (`.defaultCode`) because default accounts lack MP configurations, ship-to bindings, and MP inventory.
  - Ensure the target market/role fixture in `integration_test/data/users/{market}/{role}.json` contains the `"marketPlace"` block under `"customerAndShipTo"`.
- **Omit Consumptive Terminal Submits**: In transactional modules (**Payment Advice**, **Return Requests**), omit the final submit action to preserve finite reference data (open invoices, order lines). Assert up to Review screen or confirmation modal.
- **No Shared Config Mutation**: Never mutate global or Sales Organization default configurations in back-office portals.
- **Legacy Code Protection**: Strictly forbidden to clean up, refactor, or delete unrelated legacy code/references unless explicitly instructed.

---

## 3. Step-by-Step Implementation Workflow

### Step 1: Zephyr Preconditions & Folder Gate
1. **Status Check**: Status in Zephyr Scale MUST be **`Approved`**. If `Draft` or `Review`, STOP immediately and warn user.
2. **Transition Status**: Transition test case to **`Automating`** upon pickup.
3. **Folder Location Gate**:
   - Verify the test case is located under `Automation Tests/...` (Folder ID: `47863388`).
   - If located outside (e.g. `Manual Tests`, `Regression Test Suite`), **STOP and WARN user immediately** with the detected breadcrumb.
4. **Platform Check**: Ensure `Platform` custom field includes `Mobile` or `All Platforms`.
5. **Technical Blocker Routing**: If a blocker is discovered (untestable 3rd-party OTP, volatile data): transition status to **`Can't Automate`**, add blocker comment, and move to `Manual Tests (47863438)`.

### Step 2: Mirror Zephyr Folder Structure (Flat 1:1 Rule)
The code path under `integration_test/tests/` is a **flat 1:1 mirror** of the Zephyr folder chain under `Automation Tests`.
- The **leaf folder becomes the test file name** (`<leaf_snake>_test.dart`) directly under its parent. Do NOT create extra subfolders.
- Example: `Automation Tests/Products/Product Listing` -> `integration_test/tests/products/product_listing_test.dart`.
- Example: `Automation Tests/Checkout and Order/Item Detail` -> `integration_test/tests/checkout_and_order/item_detail_test.dart`.

### Step 3: Configure User Test Data
- Configure credentials and fixtures in `integration_test/data/users/{market}/{role}.json`. Never hardcode in test scripts.
- **Matcode Source Discipline**: Tracing historical matcodes:
  ```bash
  git log --all -S '"<fieldName>": "<matcodePrefix>"' -- integration_test/data/users/<market>/
  ```
  Never copy matcodes across distinct fields (e.g. `plant2027List.first` != `dropShippingMaterial`). If missing, ask user.
- **Schema Extension**: Update (1) `JsonSessionTestData` class + `empty`, (2) `data_loader.dart` parser, and (3) market JSON fixtures in lockstep.

### Step 4: Audit Robots & Map Steps
1. Audit existing methods in `integration_test/robots/` via `locator<RobotName>()`. Never guess names.
2. Add missing methods to Robot classes following existing patterns.
3. **Zero Conditional Expectations**: Robot methods must NEVER contain `if` conditions that skip `expect(...)`. All assertions must be deterministic hard checks.

### Step 5: Implement Test & Register in Central Runner
1. Implement test in `integration_test/tests/<feature>/<name>_test.dart`:
   - Follow pattern from [product_listing_test.dart](file:///Users/sotatek/Desktop/eZRx-Mobile.worktrees/huannt/integration_test/tests/products/product_listing_test.dart).
   - Use `testCaseWrapper` with filtered `markets: const [...]` and `roles: const [...]`.
2. Register in `integration_test/tests/runner_test.dart`:
   - Instantiate test class with `JsonSessionTestData.empty()`.
   - In `setUpAll()`, call `TestClass.useTestData(jsonSessionTestData)`.
   - Register test method in matching `group('<Domain> - <Feature>', () { ... })`.

---

## 4. Execution, Debugging & CI/CD Gate

### Local Execution (Patrol + Android Emulator Only):
Always use Android emulator, never iOS simulator:
```bash
# Run specific test code(s) for target market and role
patrol test -t integration_test/tests/runner_test.dart \
  -d <device_id> \
  --dart-define MARKET=<market> \
  --dart-define ROLE=<role> \
  --dart-define TESTCODES=<EZRX-TXXXX>
```

### Debugging & 2-Retry Stop Gate:
- Use `--show-flutter-logs` and temporary `print('[IT_DEBUG] ...')` (remove before commit).
- **2-Retry Rule**: NEVER retry a failed test more than **2 times total** (initial + 1 retry). After 2nd failure: STOP immediately, summarize error/suspected cause, and ask user. Do NOT modify code or credentials autonomously.

### PR & Pipeline Verification:
1. Run Azure Pipeline: `Pipelines > Regression Check Automation > Run pipeline`.
2. Set branch, markets, roles, and whitelist test codes (`TESTCODES`).
3. **Merge Gate**: Pipeline execution MUST achieve **100% PASS** on the `MB Automation Regression` MS Teams channel report before PR approval.
4. **Post-Pass Status**: Update Zephyr status to **`Automated MB`** (or **`Automated`** if Web FE is also done).
