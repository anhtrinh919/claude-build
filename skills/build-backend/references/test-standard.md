# Test Standard — tests that catch bugs, not just pass CI

Required reading for `/build-backend` before writing any test (Stage 1). Also applies whenever writing, reviewing, or expanding a test file elsewhere in the `/build` stack.

**Success bar:** the mechanism under test is never mocked; every assertion names a specific value; every regression test fails before the fix and passes after; the feature is exercised on a real path before it's called done.

## State the behavior before touching code

One sentence per behavior being proven: given this input/action, this observable outcome happens. If the expected outcome can't be stated without referencing the implementation, the test is proving what the code does, not whether it should do it.

- Bad: "calls `formatUser()` with the user object"
- Good: "returns 'John D.' for a user with first='John' last='Doe'"

## Test layer hierarchy

Write at the **lowest layer that can actually catch the failure**:

1. **Integration** — real collaborations between modules, real (or close-to-real) dependencies. Most bugs live here: units that work alone but compose incorrectly.
2. **Unit** — pure logic, transformations, business rules with no external dependency. Fast, valuable for complex algorithms.
3. **E2E / dogfood** — critical full user flows that can't be verified lower down. Sparingly — slow, brittle, and owned by `/build-review`, not this skill.

Default to integration tests for any code touching multiple modules, the DB, an API, or state management.

## Mock discipline

Mocking the layer where the bug lives is the most common reason a suite passes while the feature is broken.

**Mock only:** external services (Stripe, SendGrid), slow I/O in unit tests, time and randomness.

**Never mock:** the module whose correctness is being proven, or security/crypto internals that are the *mechanism* under test. When the code's job is to call `jwt.verify`, `bcrypt.compare`, or `crypto.createHmac` correctly, mocking it proves nothing — a mock returning `{ userId: 1 }` passes even if the real token is expired, tampered, or signed with the wrong secret.

**Auth middleware tests:** generate a real token with the same library, sign it with a test secret, let the real `verify` run.

```typescript
// correct — real verification happens
const TEST_SECRET = 'test-secret-do-not-use-in-prod';
const validToken = jwt.sign({ userId: 'user-1' }, TEST_SECRET, { expiresIn: '1h' });
const expiredToken = jwt.sign({ userId: 'user-1' }, TEST_SECRET, { expiresIn: '-1s' });
const wrongSecretToken = jwt.sign({ userId: 'user-1' }, 'wrong-secret');
process.env.JWT_SECRET = TEST_SECRET;
// call the middleware with these real tokens — no jwt mock needed

// wrong — proves nothing about real JWT verification
jest.mock('jsonwebtoken');
mockJwt.verify.mockReturnValue({ userId: 'user-1' }); // passes even if secret is wrong
```

**Integration seams (DB, internal services):** prefer real — in-memory DB, test container, local DB. If the environment can't support that, mocking is acceptable but document the gap and match the real wire shape, not just the type (`{ id: 1 }` vs. the real `{ id: "uuid-string" }` is a lying mock).

## Assertion quality

For every assertion: if a realistic bug were introduced, would this catch it? `toBeTruthy()` is never sufficient.

Bad: `expect(result).toBeTruthy()` · `expect(arr.length).toBeGreaterThan(0)` · `expect(typeof result).toBe('object')`

Good: `expect(result).toBe('John D.')` · `expect(arr).toHaveLength(3)` · `expect(result).toEqual({ id: 42, status: 'confirmed' })`

Assert on **outcomes**, not mechanics — `expect(emailService.send).toHaveBeenCalledWith(user.email)` breaks on refactor and passes on behavioral regression; `expect(await getLastEmail(user.email)).toMatchObject({ subject: 'Order confirmed' })` doesn't.

## Regression tests: fail first, then fix

1. Write the test against *unfixed* code — it must fail. This proves it's a real regression anchor.
2. Fix the implementation.
3. Confirm the test passes.

A test written after the fix that only ever passes is documentation, not a regression anchor. If reported inputs don't crash, don't conclude "no bug" — test the *adjacent* inputs: null/undefined from form fields, empty arrays, zero values, a different code path.

## Required edge cases

For every behavior under test, cover — or state explicitly why a category doesn't apply:

- **Empty:** `[]`, `""`, `null`, `undefined`, `{}`
- **Zero and false:** valid values, not the same as missing
- **Boundary:** exactly at the limit (max string length, first/last element, exactly-equal thresholds)
- **Error body on 200:** `{ success: false, error: null }` with HTTP 200 — does the code treat this as success or failure?
- **Real-world shape:** `name` null? `id` a UUID string, not an integer? array with 10,000 items?
- **Post-failure state:** what happens on the second attempt after the first one failed?

## Snapshot tests

Never run `--updateSnapshot` to make a failing test pass without reading the diff and confirming the change is intentional. A snapshot failure is a bug report — investigate first. If the reason a snapshot changed can't be determined, don't update it.

## What "done" actually means

Tests passing + typecheck clean = code compiles and tests are internally consistent. It does **not** mean the feature works for a real user. Before marking a group complete: exercise the actual user path, not the test path, and verify the observable outcome matches what was asked for. If the environment is headless, say so and give exact steps for the user to verify. Never say "done" on CI alone.

**The false-completion trap:** when a test fails, fix the code, don't weaken the assertion. Weakening converts a bug-detector into dead weight — "I'll loosen this for now" is the signal to stop and understand the failure, not silence it.
