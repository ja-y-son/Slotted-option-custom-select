# Slotted Options in Custom Select

## What is the problem?

The `<select>` element does not support slotted `<option>` elements. When a `<select>` lives inside a shadow root and `<option>` elements are slotted in from the light DOM, the select does not recognize them — it traverses the regular DOM tree, not the flat tree.

This is **critical for web components**. Design systems are overwhelmingly built with web components today, yet there is no way to wrap a native `<select>` in a custom element and let consumers provide `<option>` elements via slots. The only workaround is to rebuild the entire select from scratch using MutationObservers, custom DOM, heavy JS, and manual accessibility — exactly the kind of platform gap web components were meant to close.


**Reference:** [whatwg/html#11535](https://github.com/whatwg/html/issues/11535)

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
<!-- The select does NOT see the slotted options -->
```
### Nested case

### Do we want to support case?
select element get slotted into an inner shadow root and there's an option underneath it


## Additional requirement
- Performant
- Does not affect old code path, only applies to base-appearance custom select


## Non-goal:
- Allowing options slooted into non-base-appearance select

## What are we missing in HTML/DOM spec?

## Design:
- The correct select needs to know it needs to recalc
    - Can base-appearance select know that its base-appearance?
- Who’s the postman? Someone needs to let select know that it needs recalc
    - Shadow root telling select to recalc?
        - how to correctly target the select instead of telling all select to recalc
    - Slot element telling select to recalc? (After assigned)
        - how to handle nested