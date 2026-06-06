# 07 — Accessibility

The upstream community guide is silent on accessibility; this overlay is not, because a control a user cannot reach is a control that does not work. REACT-4 states it plainly: accessibility is correctness — the first value the core guide opens with (correctness > performance > DX), applied to the human at the edge instead of the type at the boundary. This chapter is not compliance theater — not a WCAG checkbox chased the week before audit — but whether the feature functions for everyone who arrives at it: by keyboard, by screen reader, by switch device, by a screen too small or eyes that parse contrast differently. A11y bugs are correctness bugs, so they are caught the way every correctness bug is: by the linter (7.4) for the static half and by tests (7.7) for the rest. Neither is optional, and an inaccessible control fails review like any other broken behaviour.

## What good looks like

```tsx
function EmailField({onSubmit}: {onSubmit: (email: string) => void}) {
  const [email, setEmail] = useState('');
  const [error, setError] = useState<string | null>(null);
  const inputRef = useRef<HTMLInputElement>(null);
  function handleSubmit(event: React.FormEvent) {
    event.preventDefault();
    const problem = validateEmail(email);
    setError(problem);
    if (problem) inputRef.current?.focus(); // 7.2: focus moves to the field to fix
    else onSubmit(email);
  }
  return (
    <form onSubmit={handleSubmit} noValidate>
      <label htmlFor="email">Email address</label> {/* 7.1, 7.3: real label, wired by htmlFor */}
      <p id="email-hint">We use this only to send your receipt.</p>
      <input
        ref={inputRef} id="email" type="email" value={email}
        onChange={(e) => setEmail(e.target.value)}
        aria-describedby={error ? 'email-hint email-error' : 'email-hint'}
        aria-invalid={error != null} // 7.3: state announced, not just recolored (7.6)
      />
      {error && <p id="email-error" role="alert">{error}</p>} {/* 7.6: text, not color alone */}
      <button type="submit">Save email</button> {/* 7.1, 7.2: native, keyboard-reachable */}
    </form>
  );
}
```

A `<label htmlFor="email">` binds to the input's `id`, so a click on the label focuses the field and a screen reader announces the name (7.1, 7.3). The hint and the error are wired through `aria-describedby`, which lists both ids when the field is invalid (7.3); `aria-invalid` flags the error state programmatically rather than by color (7.3, 7.6), and `role="alert"` makes the message announce the moment it appears. On a failed submit, focus moves to the offending input (7.2) instead of stranding the user wherever they were. The submit control is a native `<button>` inside a `<form>`, so Enter and Space work and no ARIA is needed (7.1). Nothing here is decoration — every attribute carries function a sighted mouse user gets implicitly.

## Rules

### 7.1 — Reach for semantic HTML first; ARIA is the fallback, never the default.

**Reasoning, step by step:**
1. Native elements arrive with behaviour, state, and a role the platform already implements and keeps correct across browsers and assistive tech. A `<button>` is focusable, fires on Enter and Space, exposes the `button` role, and reflects `disabled` — for free. A `<nav>`, a `<main>`, a `<label>`, a `<ul>` each carry meaning the accessibility tree reads directly. Rebuilding any of that on a `<div>` means reimplementing focus, keyboard handling, and roles by hand, and you will get an edge case wrong.
2. ARIA exists to fill the gaps native HTML leaves, not to paper a role onto the wrong element. The first rule of ARIA is its own warning: if a native element with the semantics you need exists, use it instead of repurposing a `<div>` with `role`. The W3C states it as "no ARIA is better than bad ARIA" — a wrong or stale ARIA attribute actively lies to a screen reader, which is worse than a plain element with no semantics at all. So `<div role="button" onClick>` is a defect on sight: it has no keyboard handler, no focusability, no Space/Enter, and it announces a role it cannot back up.

**Worked example:**
```tsx
<div role="button" onClick={onClose}>Close</div>   {/* Banned: no keyboard, no focus, a lie to AT */}
<button type="button" onClick={onClose}>Close</button> {/* Good: behaviour, role, keyboard for free */}
```
**Enforcement:** review rejects `role` on an element a native tag would cover; `jsx-a11y` (7.4) flags interactive handlers on non-interactive elements.

### 7.2 — Every interactive element is keyboard-reachable and visibly focused.

**Reasoning, step by step:**
1. A pointer is one input device among many; keyboard, switch, and screen-reader navigation all traverse the page through focus. If a control cannot be reached by Tab and operated by Enter or Space, it does not exist for those users — the feature is as broken as if it threw. The check is mechanical and non-negotiable: tab through every new piece of UI before requesting review, confirming each control is reachable, operable, and that focus order follows reading order. Native elements (7.1) give you this for free; custom widgets must earn it with `tabIndex`, key handlers, and the right role.
2. The focus ring is the keyboard user's cursor — the only signal of where they are. `outline: none` with nothing in its place deletes that cursor; it removes a function, not a style, and is banned on the same footing as deleting the control. If the default ring clashes with the design, replace it with a `:focus-visible` style of equal or better clarity; never remove it outright. Focus must also be *managed*, not just preserved: a dialog traps focus on open and returns it to the trigger on close; a validation error moves focus to the field that failed (see the exemplar).

**Worked example:**
```css
/* Banned: deletes the keyboard user's cursor */
button:focus { outline: none; }
/* Good: a clear, deliberate focus indicator */
button:focus-visible { outline: 2px solid var(--focus); outline-offset: 2px; }
```
**Enforcement:** tab-through is a review gate for every new view; `outline: none` without a `:focus-visible` replacement is rejected; `jsx-a11y` flags positive `tabIndex` and missing key handlers.

### 7.3 — Every input has a programmatic label; errors wire through `aria-describedby` and `aria-invalid`.

**Reasoning, step by step:**
1. An input without a programmatically associated label is anonymous to a screen reader — it announces "edit text, blank," and the user has no idea what to type. The label must be *associated*, not merely nearby: a visible `<label htmlFor={id}>` bound to the input's `id`, or `aria-labelledby` pointing at existing visible text. A placeholder is not a label: it vanishes the instant the user types, fails contrast in every browser default, is inconsistently announced, and leaves no prompt once a value is present. It is at most a supplementary example, never the name of the field.
2. Errors and hints must reach the same accessibility tree as the value. Wire help text and validation messages with `aria-describedby` (it accepts a space-separated list of ids, so a hint and an error coexist, as in the exemplar), and mark the invalid state with `aria-invalid` so it is announced as state, not inferred from a red border (7.6). Put the error in a live region — `role="alert"` or `aria-live="assertive"` — so it is spoken the moment it renders, not only when the user navigates back to the field.

**Worked example:**
```tsx
<input placeholder="Email" /> {/* Banned: placeholder is gone on focus, never the name */}
// Good: real label, described-by hint + error, invalid state exposed (full form in the exemplar)
<label htmlFor="e">Email</label>
<input id="e" aria-describedby="e-err" aria-invalid={hasError} />
{hasError && <p id="e-err" role="alert">Enter a valid email.</p>}
```
**Enforcement:** `jsx-a11y/label-has-associated-control` requires a bound label; review forbids placeholder-as-label and requires `aria-describedby` + `aria-invalid` on validated fields.

### 7.4 — Run `eslint-plugin-jsx-a11y` in the overlay as the mechanical floor.

**Reasoning, step by step:**
1. A large class of accessibility defects is static and therefore machine-detectable: an `<img>` with no `alt`, a `<label>` bound to nothing, an `onClick` on a `<div>` with no keyboard handler, a positive `tabIndex`, an invalid `aria-*` attribute or value. Catching these in the editor is strictly cheaper than catching them in review or in production, so the overlay runs `eslint-plugin-jsx-a11y` with its `recommended` config alongside `eslint-plugin-react-hooks` (REACT-2). It belongs in the same correctness-justified tier as the hooks plugin: not a style preference but a lint that encodes the Rules of React's a11y obligations, and like `exhaustive-deps` its findings are errors, fixed at the source rather than suppressed.
2. The plugin is a floor, not a ceiling, and the boundary is sharp: it verifies the *static* shape of the JSX — that attributes exist and are well-formed — and is blind to everything dynamic. It cannot tab through your UI, cannot tell whether focus returns to the trigger when a dialog closes, cannot judge whether an `alt` string is meaningful or whether color is the only signal. Linting catches the static half; the human (7.2) and the test suite (7.7) catch the flow. Passing `jsx-a11y` clean is the entry condition for review, never its conclusion.

**Worked example:**
```js
// eslint.config.js (flat) — a11y lint sits beside the hooks lint, both correctness-tier
import jsxA11y from 'eslint-plugin-jsx-a11y';
import reactHooks from 'eslint-plugin-react-hooks';
export default [jsxA11y.flatConfigs.recommended, reactHooks.configs['recommended-latest']];
```
**Enforcement:** `jsx-a11y/recommended` runs in CI with violations as errors; suppressions require a written justification in review, the same bar as silencing `exhaustive-deps`.

### 7.5 — Images, icons, and media carry text alternatives.

**Reasoning, step by step:**
1. A screen reader cannot see a graphic; the text alternative is the only thing it can announce, so every non-text element must declare what it is — including declaring that it is nothing. An *informative* image needs an `alt` that conveys its information as the content it replaces, not as "image of": a chart's `alt` states the trend, an avatar's `alt` is the person's name. A *decorative* image — a divider, a flourish, an icon that merely repeats adjacent text — declares itself with `alt=""` (an explicit empty string, not a missing attribute), which removes it from the accessibility tree so the screen reader skips it cleanly. A missing `alt` is the bug: assistive tech then reads the file name aloud.
2. Icon-only controls are the most common silent failure. A button whose entire content is an SVG has no accessible name — it announces "button," nothing more. Give it an `aria-label` (or visually hidden text) for the action, and mark the decorative glyph inside it `aria-hidden`. Time-based media carries the same obligation: video needs captions, audio needs a transcript, because the content is otherwise unreachable.

**Worked example:**
```tsx
<img src="/chart.png" alt="Revenue rose 40% from Q1 to Q2" /> {/* informative: describe it */}
<img src="/divider.svg" alt="" />                            {/* decorative: declare emptiness */}
<button aria-label="Delete row"><TrashIcon aria-hidden="true" /></button> {/* icon-only: named */}
```
**Enforcement:** `jsx-a11y/alt-text` requires an `alt` decision on every image; review verifies icon-only controls have an `aria-label` and that `alt` strings are meaningful, not filenames.

### 7.6 — Color is never the only signal.

**Reasoning, step by step:**
1. Color carries no information to a user who cannot distinguish it — color-blind, low-vision, on a washed-out screen in sunlight, or listening through a screen reader that has no concept of red. So color must always be *redundant*: pair it with text, an icon, a shape, or a pattern that carries the same meaning on its own. A required field marked only by a red asterisk-colored label is invisible to those users; add the word "required" or an icon. An error shown only as a red border fails the same way — which is why 7.3 mandates `aria-invalid` and a text message, so the state is announced and read, not merely tinted. A status that changes must announce the change (a live region), not just recolor a dot.
2. When color *is* used, it must clear the contrast bar so low-vision users can perceive it at all. Text and its background meet WCAG AA: a 4.5:1 ratio for normal text, 3:1 for large text and for the visual boundaries of UI components and meaningful graphics. This is a measurable threshold, checked with a contrast tool against the design tokens, not eyeballed. The two halves compose: meaning never rides on hue alone, and the hues you do use are perceivable.

**Worked example:**
```tsx
// Banned: the only cue is color — invisible to a color-blind or screen-reader user
<span style={{color: 'red'}}>Failed</span>
// Good: icon + text carry the meaning; color is reinforcement, and contrast meets AA
<span className="status-error"><XIcon aria-hidden="true" /> Failed</span>
```
**Enforcement:** review requires a non-color cue (text/icon/pattern) for every state; contrast is verified against AA (4.5:1 text) via a contrast checker on the tokens; state changes route through a live region (7.3).

### 7.7 — Role-query testing is the enforcement loop.

**Reasoning, step by step:**
1. Testing Library's role queries traverse the same accessibility tree a screen reader does, which makes them a proxy for reachability rather than a mere convenience. `getByRole('button', {name: 'Save email'})` finds the control only if it exposes the `button` role *and* an accessible name — exactly the two things a screen reader needs and the two things the rules above provide. The contrapositive is the whole point: if `getByRole` cannot find an element, neither can assistive technology. A test written this way (the default mandated by chapter [06](./06-testing-react.md) §6.1) does not merely check behaviour; it asserts accessibility as a side effect, so a missing label or a `<div>`-button fails a test instead of slipping to production.
2. This closes the loop the other rules open. The linter (7.4) catches the static half; role-query tests catch the dynamic half the linter cannot see — that focus moved to the errored field, that the dialog trapped and restored focus, that the error landed in a live region and was announced. Because these assertions run on every push (chapter [06](./06-testing-react.md)), an accessibility regression turns a build red the same day it is written, which is what it means to enforce a11y in tests rather than discover it in a quarterly audit. The audit finds what already shipped; the test refuses to let it ship.

**Worked example:**
```tsx
it('focuses the email field and announces the error on invalid submit', async () => {
  render(<EmailField onSubmit={vi.fn()} />);
  const user = userEvent.setup(); // 06 §6.2: setup() once per test
  await user.click(screen.getByRole('button', {name: 'Save email'})); // found only if accessible
  expect(screen.getByRole('alert')).toHaveTextContent(/valid email/i); // announced (7.3)
  expect(screen.getByRole('textbox', {name: 'Email address'})).toHaveFocus(); // managed (7.2)
});
```
**Enforcement:** components ship with role-query tests per chapter [06](./06-testing-react.md) §6.1; a control unreachable by `getByRole` fails review; focus and live-region behaviour are asserted, not left to manual audit.

## Cross-references

- Accessibility is correctness, the overlay's framing of the correctness value for the human at the edge: [README.md](./README.md) (REACT-4).
- Native elements over `role`-on-`div` and controlled inputs: [01-components-and-props.md](./01-components-and-props.md); composition over a forest of boolean props: [README.md](./README.md) (REACT-6).
- Focus management in effects, cleanup, and the Rules of Hooks the focus-trap relies on: [02-hooks.md](./02-hooks.md) and [README.md](./README.md) (REACT-2, REACT-5).
- `react-hook-form` + `zod` resolvers, where the labels, `aria-describedby`, and error wiring of 7.3 live in real forms: [04-data-fetching-and-forms.md](./04-data-fetching-and-forms.md).
- Role-based queries, `user-event` over `fireEvent`, behaviour-first testing — the enforcement loop of 7.7 — and the assertion-as-correctness discipline `jsx-a11y` joins: [06-testing-react.md](./06-testing-react.md) (§6.1) and core [11-testing.md](../typescript/11-testing.md).
