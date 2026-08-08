# Test Standard — tests that catch bugs, not just pass CI

Required reading for `/build:backend` before it writes any test (Step 1). It also applies whenever you write, review, or expand a test file anywhere in the `/build` stack.

**Success bar:** the mechanism under test is never mocked; every assertion names a specific value; every regression test fails before the fix and passes after; the feature runs on a real path before anyone calls it done.

## State the behavior before touching code

One sentence per behavior: given this input or action, this observable outcome happens. If you cannot state the expected outcome without naming the implementation, the test proves what the code does, not whether it is right.

- Bad: "calls `formatUser()` with the user object"
- Good: "returns 'John D.' for a user with first='John' last='Doe'"

## Test layer hierarchy

Write at the **lowest layer that can catch the failure**:

1. **Integration** — real collaborations between modules, with real or close-to-real dependencies. Most bugs live here: units that work alone but compose incorrectly.
2. **Unit** — pure logic, transformations, and business rules with no external dependency. Fast, and valuable for complex algorithms.
3. **E2E / dogfood** — critical full user flows that no lower layer can verify. Use sparingly; `/build:review` owns them, not this skill.

Default to integration tests for any code that touches multiple modules, the DB, an API, or state management.

## Mock discipline

Mocking the layer where the bug lives is the most common reason a suite passes while the feature is broken.

**Mock only:** external services (Stripe, SendGrid), slow I/O in unit tests, time, and randomness.

**Never mock** the module whose correctness you are proving, or the security and crypto internals that are the *mechanism* under test. When the code's job is to call `jwt.verify`, `bcrypt.compare`, or `crypto.createHmac` correctly, a mock returning `{ userId: 1 }` passes even if the real token is expired, tampered with, or signed with the wrong secret.

**Auth middleware tests:** generate real tokens with the same library, sign them with a test secret, and let the real `verify` run. Cover the valid, expired, and wrong-secret cases.

```typescript
const TEST_SECRET = 'test-secret-do-not-use-in-prod';
const validToken       = jwt.sign({ userId: 'user-1' }, TEST_SECRET, { expiresIn: '1h' });
const expiredToken     = jwt.sign({ userId: 'user-1' }, TEST_SECRET, { expiresIn: '-1s' });
const wrongSecretToken = jwt.sign({ userId: 'user-1' }, 'wrong-secret');
process.env.JWT_SECRET = TEST_SECRET;
```

**Integration seams (DB, internal services):** prefer real — an in-memory DB, a test container, a local DB. Where the environment cannot support that, a mock is acceptable, but document the gap and match the real wire shape, not just the type. `{ id: 1 }` against a real `{ id: "uuid-string" }` is a lying mock.

## Assertion quality

For every assertion, ask: if a realistic bug were introduced, would this catch it? `toBeTruthy()` never is.

Bad: `expect(result).toBeTruthy()` · `expect(arr.length).toBeGreaterThan(0)` · `expect(typeof result).toBe('object')`

Good: `expect(result).toBe('John D.')` · `expect(arr).toHaveLength(3)` · `expect(result).toEqual({ id: 42, status: 'confirmed' })`

Assert on **outcomes**, not mechanics. `expect(emailService.send).toHaveBeenCalledWith(user.email)` breaks on a refactor and passes on a behavioral regression; `expect(await getLastEmail(user.email)).toMatchObject({ subject: 'Order confirmed' })` does neither.

## Regression tests: fail first, then fix

1. Write the test against the *unfixed* code. It must fail.
2. Fix the implementation.
3. Confirm the test passes.

A test written after the fix, which only ever passes, is documentation, not a regression anchor. If the reported inputs do not crash, do not conclude "no bug" — test the *adjacent* inputs: null or undefined from form fields, empty arrays, zero values, a different code path.

## Required edge cases

For every behavior under test, cover each category or state why it does not apply:

- **Empty:** `[]`, `""`, `null`, `undefined`, `{}`
- **Zero and false:** valid values, not the same as missing
- **Boundary:** exactly at the limit — max string length, first and last element, exactly-equal thresholds
- **Error body on 200:** `{ success: false, error: null }` with HTTP 200 — does the code treat this as success or failure?
- **Real-world shape:** `name` null? `id` a UUID string, not an integer? an array with 10,000 items?
- **Post-failure state:** what happens on the second attempt after the first one failed?

## Snapshot tests

Never run `--updateSnapshot` to make a failing test pass before you read the diff and confirm the change is intentional. A snapshot failure is a bug report. If you cannot determine why a snapshot changed, do not update it.

## What "done" actually means

Tests passing plus a clean typecheck means the code compiles and the tests are internally consistent. It does **not** mean the feature works for a real user. Before you mark a group complete, exercise the actual user path, not the test path, and verify the observable outcome matches what was asked for. In a headless environment, say so and give exact steps for the user to verify. Never say "done" on CI alone.

**The false-completion trap:** when a test fails, fix the code — never weaken the assertion. "I'll loosen this for now" is the signal to stop and understand the failure, not to silence it.
