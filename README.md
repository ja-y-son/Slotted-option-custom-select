# Slotted Options in Custom Select

## What is the problem?

The `<select>` element does not support slotted `<option>` elements. When a `<select>` lives inside a shadow root and `<option>` elements are slotted in from the light DOM, the select does not recognize them — it traverses the regular DOM tree, not the flat tree.

This is **critical for web components**. Design systems are overwhelmingly built with web components today, yet there is no way to wrap a native `<select>` in a custom element and let consumers provide `<option>` elements via slots. The only workaround is to rebuild the entire select from scratch using MutationObservers, custom DOM, heavy JS, and manual accessibility — exactly the kind of platform gap web components were meant to close.

**Reference:** [whatwg/html#11535](https://github.com/whatwg/html/issues/11535)

## Scope
This change **only** applies to `<select>` elements with `appearance: base-select`. Non-base-appearance selects are unaffected.

## Code examples

### Common case

```html
<my-custom-select>
  <template shadowrootmode=open>
    <select>
      <slot></slot>
    </select>
  </template>
  <option>one</option>
  <option>two</option>
</my-custom-select>
```

**Expected:** The `<select>` should recognize both slotted `<option>` elements as its own options and render them in its dropdown. All imperative APIs (see [Additional requirements](#additional-requirement)) should work as if the options were direct children.
### Nested case

Options are in the light DOM of `<my-section>`, slotted through its `<slot>`, then forwarded through `<my-custom-select>`'s `<slot>` into the inner `<select>`. This involves multi-level slot forwarding across two shadow boundaries.

```html
<my-section>
    <template shadowrootmode=open>
        <my-custom-select>
            <template shadowrootmode=open>
                <select>
                    <slot></slot>
                </select>
            </template>
        </my-custom-select>
        <slot></slot>
    </template>
    <option>one</option>
    <option>two</option>
</my-section>
```

**Expected:** The `<select>` should still recognize the options despite traversing multiple shadow boundaries. This must work — design system components are commonly nested.

### Slotted button

In base-appearance select, the first child `<button>` element is treated as the select's trigger button, replacing the default rendering. When both a `<button>` and `<option>` elements are slotted in through the same `<slot>`, the select must correctly distinguish them: the `<button>` should be recognized as the trigger, and the `<option>` elements should be recognized as options.

```html
<my-custom-select>
    <template shadowrootmode=open>
        <select>
            <slot></slot>
        </select>
    </template>
    <button>New button</button>
    <option>one</option>
    <option>two</option>
</my-custom-select>
```

**Expected:** `select.options` returns the two options (not the button). The button replaces the default select trigger.


## Additional requirements
- Slotted `<optgroup>` elements (with nested `<option>` children) should be supported and behave the same as direct-child optgroups — grouping is preserved, `disabled` on the optgroup disables its options, etc.
- All imperative capabilities of select element, such as `select.options`, `select.selectedOptions`, `select.selectedIndex`, etc, should work seamlessly and treat all the slotted-in options as if they're light DOM children of select element.
- Performance: A full flat-tree walk on every option query is too expensive. The implementation must be performant, but the specific approach is open to exploration.
- The select's option list must stay in sync as options/optgroups are added, removed, or re-slotted.

## Current implementation ideas
- **Notification-based approach:** React to slot assignment changes (e.g., `slotchange` events or the spec's "signal a slot change" mechanism) to maintain or invalidate a cached option list, rather than re-walking the tree on every access.
- The select would need to recalculate its option list when:
  - An `<option>` or `<optgroup>` is inserted into or removed from the light DOM
  - Slot assignment changes, which may add or remove options from the select's flat tree
  - An `<option>` is added to or removed from a slotted `<optgroup>`

## What are we missing in HTML/DOM spec?

The following spec algorithms and definitions assume DOM tree traversal and would need changes to support slotted options in base-appearance select:

1. **"List of options" algorithm ([§4.10.7](https://html.spec.whatwg.org/multipage/form-elements.html#concept-select-option-list))** — Walks `select`'s DOM children/descendants in tree order. Slotted options are not DOM descendants of the select element, so they are never found. This algorithm needs to traverse the flat tree (or equivalent) for base-appearance selects.

2. **"Option element nearest ancestor select" ([§4.10.10](https://html.spec.whatwg.org/multipage/form-elements.html#the-option-element))** — Walks the option's DOM ancestors in reverse tree order looking for a `<select>`. A slotted option's DOM ancestors are the custom element host and its ancestors — the `<select>` in the shadow root is never encountered. This needs to walk flat-tree ancestors instead.

3. **Option insertion/removal/moving steps ([§4.10.10](https://html.spec.whatwg.org/multipage/form-elements.html#the-option-element))** — These run "update an option's nearest ancestor select," which relies on #2 above. When an option is added to the light DOM of a custom element, these steps fire but won't find the shadow DOM select. They need to account for the flat-tree relationship.

4. **`HTMLOptionsCollection` rooting ([§4.10.7](https://html.spec.whatwg.org/multipage/form-elements.html#the-select-element))** — The `options` IDL attribute returns a collection "rooted at the select node" filtered by the list of options. Since slotted options are not DOM descendants of select, the collection's rooting/filtering needs to accommodate flat-tree descendants.

5. **No slot-change hook for select ([DOM spec: "signal a slot change"](https://dom.spec.whatwg.org/#signal-a-slot-change))** — When slot assignment changes, there is currently no mechanism for a select element to be notified that options may have been added or removed from its flat tree. A new hook is needed so the select can re-run its selectedness setting algorithm when slot assignment changes.

## Prior prototyping
- [CL 6516400: Support slotting \<options\> in \<select\>](https://chromium-review.googlesource.com/c/chromium/src/+/6516400) — Adds basic functionality to slot `<option>` elements into a `<select>` by looking at the flat tree when the select has a descendant `<slot>` that fires `slotchange`. Observable via `select.options` and `select.selectedOptions`. Currently has merge conflicts and open review feedback around the overall approach (e.g., concerns about modifying `HTMLCollection` for flat-tree traversals and adding select-specific logic to `HTMLSlotElement`).
- [CL 6892064: Make \<select\> UA styles work for slotted options](https://chromium-review.googlesource.com/c/chromium/src/+/6892064) — Companion CL that changes UA stylesheet selectors for `<option>` and `<optgroup>` from ancestor-based selectors (which only match in light DOM) to pseudo-classes applied directly on the elements, enabling UA styles to work across the flat tree.
- [Design/discussion doc](https://docs.google.com/document/d/1f3dqMe00eZqC3DLDh93_D5tVRiDDByfbQDH3CU8p2gE/edit?usp=sharing)

## Design:
- The correct select needs to know it needs to recalc
    - Can base-appearance select know that its base-appearance?
- Who’s the postman? Someone needs to let select know that it needs recalc
    - Shadow root telling select to recalc?
        - how to correctly target the select instead of telling all select to recalc
    - Slot element telling select to recalc? (After assigned)
        - how to handle nested