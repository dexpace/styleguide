# 06 — Testing React

The core testing chapter ([typescript/11](../typescript/11-testing.md)) governs the doubles and the discipline: fakes over mocks (11.3), MSW at the network (11.4), determinism with no real clock or socket (11.8), and assertions in both positive and negative space (11.9). All of that holds here unchanged. The runner is the one exception. Core 11.1's family default is `bun test`, but component tests run on **Vitest** with globals off — a recorded runtime substitution, mirrored in the [README ledger](./README.md#deviations-from-upstream) against [typescript-bun](../typescript-bun/)'s runtime ledger, because `bun test` runs no Service-Worker interception for MSW (6.3) and its DOM story is still immature. This chapter is the component layer on top: how you render a component, query it the way a user reaches it, drive it through real interaction, and assert what the user sees. That is the seam where those core rules meet the DOM and the accessibility tree. It does not restate them; it cites them and shows their React shape.

## What good looks like

```tsx
import {describe, it, expect, beforeAll, afterEach, afterAll} from 'vitest';
import {render, screen} from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import {setupServer} from 'msw/node';
import {http, HttpResponse} from 'msw';
import {SubscribeForm} from './SubscribeForm.js';

const server = setupServer(
  http.post('https://api.example.com/subscribe', () => HttpResponse.json({id: 'sub_1'})),
); // the network is faked, not fetch (core 11.4)
beforeAll(() => server.listen({onUnhandledRequest: 'error'}));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('SubscribeForm', () => {
  it('confirms success and clears the error after a valid submit', async () => {
    const user = userEvent.setup();
    render(<SubscribeForm />);

    await user.type(screen.getByRole('textbox', {name: /email/i}), 'a@b.c');
    await user.click(screen.getByRole('button', {name: /subscribe/i})); // real focus→type→click (6.2)

    expect(await screen.findByRole('status')).toHaveTextContent(/subscribed/i); // bounded async (6.5)
    expect(screen.queryByRole('alert')).not.toBeInTheDocument(); // negative space: no error rendered (6.8)
  });
});
```

One component, tested as a user meets it. Every query is a role query, so the test passes only against an accessible tree (6.1, REACT-4). `userEvent` drives the real focus-then-type-then-click sequence (6.2); the network is faked at the wire by MSW, and the component never learns `fetch` exists (6.3). `findByRole` waits on a bounded default timeout rather than a sleep (6.5), and the closing assertion is negative space — proof the error region stayed empty on the happy path (6.8). Nothing asserts a class name, a hook's internal state, or a serialized tree (6.4).

## Rules

### 6.1 — Query by accessible role first; `data-testid` is the last resort.

**Reasoning, step by step:**
1. Testing Library offers query families in a deliberate priority order, and `getByRole('button', {name})` sits at the top because it reaches an element the only way a user or a screen reader can — by its role and its accessible name. A test built on role queries cannot pass against a `<div onClick>` that no assistive technology can see, so the query *is* the reachability check: it makes REACT-4 (accessibility is correctness) structural rather than aspirational, the same gate from a different seat.
2. The fallbacks descend in honesty. `getByLabelText` for form fields, `getByText` for static copy, are still user-visible. `getByTestId` is last, reserved for the element with no role and no stable text — a chart canvas, a layout wrapper — because a `data-testid` couples the test to a hook the user never perceives and that survives any amount of broken markup. Reach for it only when every accessible query is genuinely impossible, and treat its presence as a hint the component may be missing a role.

**Worked example:**
```tsx
screen.getByRole('button', {name: /save/i});      // preferred: role + accessible name
screen.getByLabelText(/email address/i);          // form fields, when no better role fits
screen.getByTestId('revenue-sparkline');          // last resort: no role, no text
```
**Enforcement:** review prefers `getByRole`/`getByLabelText`; `eslint-plugin-testing-library` flags `container.querySelector` and gratuitous `getByTestId`; a new `data-testid` is justified against the absence of any accessible query.

### 6.2 — Drive interaction with `user-event`, never `fireEvent`.

**Reasoning, step by step:**
1. `fireEvent.click(node)` dispatches one synthetic event and nothing else. A real click is a sequence: the pointer moves, the element focuses, `pointerdown`/`mousedown`/`focus`/`pointerup`/`click` fire in order, and a typed character emits `keydown`/`keypress`/`input`/`keyup`. The bugs live in the gaps `fireEvent` skips — the handler that needs focus first, the field that validates on `keydown`, the button that is disabled until blur. A test that dispatches one event passes over exactly the lifecycle where the defect hides.
2. `userEvent.setup()` replays the full browser sequence, so the test exercises the component the way the user does. Every interaction is `async` and must be awaited — the sequence yields to React between steps — and `setup()` is called once per test, never shared across tests (core 11.10). `fireEvent` is a narrow escape hatch for a low-level event `user-event` does not model (a synthetic `scroll`, a `paste` with crafted clipboard data), not the default.

**Worked example:**
```tsx
const user = userEvent.setup();
render(<LoginForm />);

await user.type(screen.getByLabelText(/password/i), 'hunter2'); // keydown→input→keyup per char
await user.tab();                                                // real focus move fires blur validation
await user.click(screen.getByRole('button', {name: /log in/i}));
```
**Enforcement:** review; `eslint-plugin-testing-library`'s `prefer-user-event`; `fireEvent` calls justified against an event `user-event` cannot produce.

### 6.3 — Route every request through MSW; never mock the API module.

**Reasoning, step by step:**
1. Component tests inherit core 11.4 verbatim: HTTP is faked at the network with Mock Service Worker, so the real `fetch` runs and the component exercises genuine URL construction, headers, status handling, and deserialization. The fake is the *server*. MSW is exactly why component tests run on **Vitest** rather than the family's `bun test` default (core [11.1](../typescript/11-testing.md)): `bun test` ships no Service-Worker/network interception, so MSW has nothing to route through, and its DOM story is still immature. That swap is the recorded runtime substitution in the [README ledger](./README.md#deviations-from-upstream); the MSW discipline below is otherwise unchanged from core. A test that instead does `vi.mock('./api')` to stub your data-access module asserts your wiring against your own assumptions and skips the layer most likely to be wrong — the one core 11.3/11.4 exist to cover.
2. Handlers are defined per feature and composed, so the success path is the suite-wide default and a single `server.use(...)` overrides one endpoint to a 500, a timeout, or an empty list for the test that needs that case. Run the server with `onUnhandledRequest: 'error'` so an unmocked call fails loudly instead of escaping to the real network and flaking. The query client wrapped around the component is configured with retries off, so a deliberate error surfaces on the first response rather than after backoff (core 11.8).

**Worked example:**
```tsx
server.use(
  http.get('https://api.example.com/orders', () =>
    HttpResponse.json({message: 'upstream down'}, {status: 500})),
); // this test only: force the error branch; the default handler stays green
render(<OrderList />, {wrapper: makeQueryWrapper({retry: false})});
expect(await screen.findByRole('alert')).toHaveTextContent(/try again/i);
```
**Enforcement:** review; no `vi.mock` of owned API modules (ports core 11.3); MSW runs with `onUnhandledRequest: 'error'`.

### 6.4 — Test behaviour the user observes, not implementation detail.

**Reasoning, step by step:**
1. A test exists to fail when behaviour breaks and stay green through every refactor that preserves it. Asserting implementation — a `useState` value, a CSS class, a child component's name, a whole serialized render tree — inverts that: the test fails when you rename a class or split a component, and passes when the visible behaviour silently regresses. Snapshot-everything is the worst form; a giant `toMatchSnapshot` asserts nothing specific, breaks on every cosmetic change, and gets blessed unread, so it catches cosmetic churn and misses real bugs.
2. Assert what the user perceives and does: text that appears, a control's enabled or checked state, a region that shows or hides, where focus lands. These are stable across internal change and meaningful when they break — exactly the property core 11.4's network seam and 11.3's fake boundaries are chosen to give you. Inline snapshots are tolerable only when scoped to one small, meaningful fragment (a formatted date string), never an entire subtree.

**Worked example:**
```tsx
// smell: couples to internals — breaks on rename, blind to behaviour
expect(container.querySelector('.btn-primary')).toBeDisabled();
// fix: assert the role and state the user and AT actually observe
expect(screen.getByRole('button', {name: /submit/i})).toBeDisabled();
```
**Enforcement:** review; no assertions on class names, component internals, or hook state; no whole-tree snapshots, only scoped inline snapshots of meaningful output.

### 6.5 — Resolve async UI with `findBy*`/`waitFor` on bounded timeouts; never sleep.

**Reasoning, step by step:**
1. Asynchronous UI — a fetch resolving, a transition committing, a toast appearing — needs the test to wait, and core 11.8's determinism mandate decides how. A fixed `await sleep(500)` is the flake generator it forbids: too short and it fails on slow CI, too long and the suite crawls, and either way the number is a guess unrelated to when the work finishes.
2. `findByRole`/`findByText` poll the DOM until the element appears or a bounded timeout elapses, and `waitFor` does the same for an arbitrary assertion. They settle the instant the condition holds, so they are both faster and steadier than any sleep, and they fail with a clear timeout instead of a hang. Keep the timeout near its default; a test that needs a long one is usually waiting on a real timer that should be virtualized with `vi.useFakeTimers()` (core 11.8) instead. Await one `findBy*` for the state you expect, then make synchronous `getBy`/`queryBy` assertions about the settled DOM — never wrap a bare `getBy` in a manual retry loop.

**Worked example:**
```tsx
await user.click(screen.getByRole('button', {name: /load more/i}));
const rows = await screen.findAllByRole('listitem'); // polls to a bounded timeout, no fixed wait
expect(rows).toHaveLength(20);
```
**Enforcement:** review; no `setTimeout`/sleep in tests; async waits go through `findBy*`/`waitFor`; long custom timeouts flag a missing fake timer (ports core 11.8).

### 6.6 — Reserve Playwright end-to-end tests for the critical flows.

**Reasoning, step by step:**
1. Component tests run against jsdom — fast, isolated, and blind to the things only a real browser has: actual navigation, real focus and tab order, third-party redirects, service workers, layout. The money paths — sign-in, checkout, the one onboarding step that must never break — cross those boundaries, so they earn a full end-to-end test in Playwright against a real browser and a real (or seeded) backend, the integration tier core 11.1 isolates under a top-level `tests/`.
2. These are deliberately few. End-to-end tests are the slowest and flakiest layer, so they cover critical journeys, not feature coverage — that load stays in component tests, where a case costs milliseconds. Each flow is owned by the feature that ships it — named for the journey it guards (`tests/checkout.e2e.test.ts`) — and lives in the isolated top-level `tests/` tier of clause 1, never colocated beside a component. It uses Playwright's own role-based locators (`getByRole`), keeping query discipline (6.1) identical across both tiers. The test pyramid stands: many component tests, a thin layer of e2e over the paths whose failure is a business incident.

**Worked example:**
```ts
test('a returning user signs in and lands on the dashboard', async ({page}) => {
  await page.goto('/login');
  await page.getByLabel(/email/i).fill('a@b.c');
  await page.getByRole('button', {name: /log in/i}).click();
  await expect(page.getByRole('heading', {name: /dashboard/i})).toBeVisible(); // real nav, real browser
});
```
**Enforcement:** review; Playwright reserved for critical flows; e2e count stays small and feature-owned; breadth lives in component tests.

### 6.7 — Assert accessibility — names, roles, focus order — in component tests.

**Reasoning, step by step:**
1. REACT-4 declares accessibility a correctness gate, and `eslint-plugin-jsx-a11y` (chapter [07](./07-accessibility.md)) catches the static mistakes — but the static linter cannot see a dialog that fails to move focus to itself, a control whose accessible name is computed wrong at runtime, or a tab order that desyncs from the visual order. Those are dynamic facts, and the component test is where the accessibility chapter's rules get teeth: the behaviour that proves the feature works for a keyboard and a screen reader, asserted, not assumed.
2. Most of it comes for free from rule 6.1 — every passing role query is already proof the element is in the accessibility tree with the right role and name. Add the dynamic assertions the queries do not cover: that opening a dialog lands focus inside it and closing returns focus to the trigger, that `Tab` visits controls in a sensible order, that a live region carries `role="status"` or `role="alert"` so updates are announced. Use `jest-dom` matchers — `toHaveAccessibleName`, `toHaveFocus` — to say it precisely. These assertions are not a separate a11y test suite; they are part of the behavioural test, because for this audience reachability *is* the behaviour.

**Worked example:**
```tsx
await user.click(screen.getByRole('button', {name: /open settings/i}));
const dialog = await screen.findByRole('dialog', {name: /settings/i});
expect(within(dialog).getByRole('button', {name: /close/i})).toHaveFocus(); // focus moved in (REACT-4)
```
**Enforcement:** review; component tests of interactive UI assert accessible names, roles, and focus movement; pairs with `eslint-plugin-jsx-a11y` (chapter [07](./07-accessibility.md)).

### 6.8 — Assert negative space in the DOM.

**Reasoning, step by step:**
1. Core 11.9 requires every test of non-trivial behaviour to assert both what must happen and what must *not*, and in the UI the negative space is its own family of assertions, easily forgotten and where the bug usually hides. "The submit succeeded" is only half a test; the other half is that the error alert did *not* render, no duplicate row appeared, and the spinner is gone. `queryByRole` is the tool — it returns `null` instead of throwing — so `expect(screen.queryByRole('alert')).not.toBeInTheDocument()` asserts an absence that `getByRole` cannot express. Use `queryBy*` only for asserting non-existence; for presence, `getBy*`/`findBy*` give better failure messages.
2. State, not only existence, has negative space. The button *is* disabled while the mutation is pending and enabled again after; the field is *not* marked invalid after a correct entry; the just-deleted item is *gone* from the list. These transient and after states are where real defects live — the double-submit because the button re-enabled too early, the error toast that lingers past the retry that fixed it. Assert the forbidden state directly, the same discipline as core 11.9 read through the DOM.

**Worked example:**
```tsx
const submit = screen.getByRole('button', {name: /pay/i});
await user.click(submit);
expect(submit).toBeDisabled();                              // negative: cannot double-submit while pending
await waitFor(() => expect(submit).toBeEnabled());
expect(screen.queryByRole('alert')).not.toBeInTheDocument(); // negative: no error after success
```
**Enforcement:** review; UI tests assert the absent error and the forbidden state via `queryBy*`/state matchers, not only the happy-path presence (ports core 11.9).

## Cross-references

- The runner, fakes over mocks, MSW at the network, determinism, and positive/negative space: core testing chapter [typescript/11](../typescript/11-testing.md) (11.1 globals-off, 11.3/11.4 doubles, 11.8 determinism, 11.9 assertion space, 11.10 no shared fixtures).
- Accessibility as a correctness gate, semantic roles, focus management, and `eslint-plugin-jsx-a11y`: chapter [07](./07-accessibility.md); the structural form of REACT-4.
- Components and props under test: chapter [01](./01-components-and-props.md); uncontrolled form inputs via react-hook-form, the shape these tests drive: chapter [04, §4.6](./04-data-fetching-and-forms.md). Hooks and dependency correctness: chapter [02](./02-hooks.md).
- Server state in TanStack Query, the cache the query wrapper configures for tests: chapter [03](./03-state-management.md) and [04](./04-data-fetching-and-forms.md).
