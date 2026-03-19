# Synthetic Event Types -- Complete API Reference

All React synthetic event types with their properties and TypeScript signatures.

---

## Base: React.SyntheticEvent\<T\>

All React events extend `SyntheticEvent`. Properties available on EVERY event:

| Property | Type | Description |
|----------|------|-------------|
| `bubbles` | `boolean` | Whether the event bubbles |
| `cancelable` | `boolean` | Whether the event can be canceled |
| `currentTarget` | `T` | The element the handler is attached to (typed by generic) |
| `target` | `EventTarget` | The element that triggered the event (may be a child) |
| `defaultPrevented` | `boolean` | Whether `preventDefault()` was called |
| `eventPhase` | `number` | Current phase (0=none, 1=capture, 2=target, 3=bubble) |
| `isTrusted` | `boolean` | Whether the event was triggered by user action |
| `nativeEvent` | `Event` | The underlying native browser event |
| `timeStamp` | `number` | Timestamp when the event was created |
| `type` | `string` | The event type name (e.g., "click", "change") |
| `preventDefault()` | `void` | Prevent the browser default action |
| `stopPropagation()` | `void` | Stop the event from bubbling to parent handlers |
| `isPropagationStopped()` | `boolean` | Whether `stopPropagation()` was called |
| `isDefaultPrevented()` | `boolean` | Whether `preventDefault()` was called |

---

## React.MouseEvent\<T\>

Handler type: `React.MouseEventHandler<T>`

| Property | Type | Description |
|----------|------|-------------|
| `altKey` | `boolean` | Alt key pressed during click |
| `button` | `number` | Which button (0=left, 1=middle, 2=right) |
| `buttons` | `number` | Bitmask of pressed buttons |
| `clientX` / `clientY` | `number` | Coordinates relative to viewport |
| `ctrlKey` | `boolean` | Ctrl key pressed |
| `metaKey` | `boolean` | Meta/Cmd key pressed |
| `movementX` / `movementY` | `number` | Movement since last mousemove |
| `pageX` / `pageY` | `number` | Coordinates relative to document |
| `screenX` / `screenY` | `number` | Coordinates relative to screen |
| `shiftKey` | `boolean` | Shift key pressed |
| `relatedTarget` | `EventTarget \| null` | Element the pointer came from/went to |
| `getModifierState(key)` | `boolean` | Check modifier key state |

### Mouse Event Props

| Prop | Bubbles | Description |
|------|---------|-------------|
| `onClick` | Yes | Click (mousedown + mouseup on same element) |
| `onDoubleClick` | Yes | Double click |
| `onMouseDown` | Yes | Mouse button pressed |
| `onMouseUp` | Yes | Mouse button released |
| `onMouseEnter` | **No** | Pointer enters element (not children) |
| `onMouseLeave` | **No** | Pointer leaves element (not children) |
| `onMouseOver` | Yes | Pointer enters element or children |
| `onMouseOut` | Yes | Pointer leaves element or children |
| `onMouseMove` | Yes | Pointer moves over element |
| `onContextMenu` | Yes | Right-click / context menu trigger |

---

## React.KeyboardEvent\<T\>

Handler type: `React.KeyboardEventHandler<T>`

| Property | Type | Description |
|----------|------|-------------|
| `key` | `string` | The key value ("Enter", "Escape", "a", "ArrowDown") |
| `code` | `string` | Physical key code ("KeyA", "Enter", "ArrowDown") |
| `altKey` | `boolean` | Alt key pressed |
| `ctrlKey` | `boolean` | Ctrl key pressed |
| `metaKey` | `boolean` | Meta/Cmd key pressed |
| `shiftKey` | `boolean` | Shift key pressed |
| `repeat` | `boolean` | Whether the key is being held down (auto-repeat) |
| `locale` | `string` | Locale string |
| `location` | `number` | Key location (0=standard, 1=left, 2=right, 3=numpad) |
| `getModifierState(key)` | `boolean` | Check modifier key state |

### Keyboard Event Props

| Prop | Description |
|------|-------------|
| `onKeyDown` | Key pressed (fires repeatedly when held) |
| `onKeyUp` | Key released |

**NEVER** use `onKeyPress` -- deprecated, does not fire for non-character keys.

---

## React.ChangeEvent\<T\>

Handler type: `React.ChangeEventHandler<T>`

No additional properties beyond `SyntheticEvent`. The value is accessed via `e.currentTarget.value`.

### Common Element Types

| Element | Generic Parameter | Access Value Via |
|---------|-------------------|------------------|
| Text input | `HTMLInputElement` | `e.currentTarget.value` |
| Checkbox | `HTMLInputElement` | `e.currentTarget.checked` |
| Radio | `HTMLInputElement` | `e.currentTarget.value` |
| Select | `HTMLSelectElement` | `e.currentTarget.value` |
| Textarea | `HTMLTextAreaElement` | `e.currentTarget.value` |
| File input | `HTMLInputElement` | `e.currentTarget.files` |

---

## React.FormEvent\<T\>

Handler type: `React.FormEventHandler<T>`

No additional properties beyond `SyntheticEvent`. Typically used with `HTMLFormElement`.

### Form Event Props

| Prop | Description |
|------|-------------|
| `onSubmit` | Form submitted (ALWAYS call `e.preventDefault()`) |
| `onReset` | Form reset |
| `onInput` | Value changed (fires immediately, before onChange) |
| `onChange` | Value changed (React fires on every keystroke for inputs) |
| `onInvalid` | Form validation failed |

---

## React.FocusEvent\<T\>

Handler type: `React.FocusEventHandler<T>`

| Property | Type | Description |
|----------|------|-------------|
| `relatedTarget` | `EventTarget \| null` | The element gaining (onBlur) or losing (onFocus) focus |

### Focus Event Props

| Prop | Bubbles | Description |
|------|---------|-------------|
| `onFocus` | Yes (in React) | Element received focus |
| `onBlur` | Yes (in React) | Element lost focus |

Note: React's `onFocus`/`onBlur` bubble, matching native `focusin`/`focusout`.

---

## React.DragEvent\<T\>

Handler type: `React.DragEventHandler<T>`

Extends `MouseEvent`. Additional property:

| Property | Type | Description |
|----------|------|-------------|
| `dataTransfer` | `DataTransfer` | Data being dragged (files, text, custom data) |

### Drag Event Props

| Prop | Fires On |
|------|----------|
| `onDrag` | Element being dragged (continuous) |
| `onDragStart` | Drag begins |
| `onDragEnd` | Drag ends (on dragged element) |
| `onDragEnter` | Dragged item enters a drop target |
| `onDragLeave` | Dragged item leaves a drop target |
| `onDragOver` | Dragged item is over a drop target (MUST call `e.preventDefault()` to allow drop) |
| `onDrop` | Item dropped on target |

---

## React.TouchEvent\<T\>

Handler type: `React.TouchEventHandler<T>`

| Property | Type | Description |
|----------|------|-------------|
| `touches` | `TouchList` | All current touch points |
| `targetTouches` | `TouchList` | Touch points on the target element |
| `changedTouches` | `TouchList` | Touch points that changed |
| `altKey` | `boolean` | Alt key pressed |
| `ctrlKey` | `boolean` | Ctrl key pressed |
| `metaKey` | `boolean` | Meta key pressed |
| `shiftKey` | `boolean` | Shift key pressed |

### Touch Event Props

| Prop | Description |
|------|-------------|
| `onTouchStart` | Finger touches the screen |
| `onTouchMove` | Finger moves on the screen |
| `onTouchEnd` | Finger leaves the screen |
| `onTouchCancel` | Touch interrupted (e.g., system gesture) |

---

## React.ClipboardEvent\<T\>

Handler type: `React.ClipboardEventHandler<T>`

| Property | Type | Description |
|----------|------|-------------|
| `clipboardData` | `DataTransfer` | Clipboard content |

### Clipboard Event Props

| Prop | Description |
|------|-------------|
| `onCopy` | Content copied |
| `onCut` | Content cut |
| `onPaste` | Content pasted |

---

## React.WheelEvent\<T\>

Handler type: `React.WheelEventHandler<T>`

Extends `MouseEvent`. Additional properties:

| Property | Type | Description |
|----------|------|-------------|
| `deltaX` | `number` | Horizontal scroll amount |
| `deltaY` | `number` | Vertical scroll amount |
| `deltaZ` | `number` | Z-axis scroll amount |
| `deltaMode` | `number` | Unit (0=pixels, 1=lines, 2=pages) |

### Wheel Event Props

| Prop | Description |
|------|-------------|
| `onWheel` | Mouse wheel scrolled |

---

## React.PointerEvent\<T\>

Handler type: `React.PointerEventHandler<T>`

Extends `MouseEvent`. Unifies mouse, touch, and stylus input. Additional properties:

| Property | Type | Description |
|----------|------|-------------|
| `pointerId` | `number` | Unique pointer identifier |
| `pointerType` | `string` | Input type: "mouse", "pen", "touch" |
| `width` / `height` | `number` | Contact geometry |
| `pressure` | `number` | Pressure (0.0 to 1.0) |
| `tangentialPressure` | `number` | Tangential pressure (-1.0 to 1.0) |
| `tiltX` / `tiltY` | `number` | Tilt angle in degrees |
| `twist` | `number` | Clockwise rotation (0 to 359) |
| `isPrimary` | `boolean` | Whether this is the primary pointer |

### Pointer Event Props

| Prop | Description |
|------|-------------|
| `onPointerDown` | Pointer activated (pressed) |
| `onPointerUp` | Pointer deactivated (released) |
| `onPointerMove` | Pointer moved |
| `onPointerEnter` | Pointer enters element (no bubble) |
| `onPointerLeave` | Pointer leaves element (no bubble) |
| `onPointerOver` | Pointer enters element (bubbles) |
| `onPointerOut` | Pointer leaves element (bubbles) |
| `onPointerCancel` | Pointer event canceled |
| `onGotPointerCapture` | Element captures pointer |
| `onLostPointerCapture` | Element loses pointer capture |

---

## Handler Type Shortcuts

React provides shorthand handler types for component props:

```tsx
type React.MouseEventHandler<T>     = (e: React.MouseEvent<T>) => void;
type React.ChangeEventHandler<T>    = (e: React.ChangeEvent<T>) => void;
type React.FormEventHandler<T>      = (e: React.FormEvent<T>) => void;
type React.KeyboardEventHandler<T>  = (e: React.KeyboardEvent<T>) => void;
type React.FocusEventHandler<T>     = (e: React.FocusEvent<T>) => void;
type React.DragEventHandler<T>      = (e: React.DragEvent<T>) => void;
type React.TouchEventHandler<T>     = (e: React.TouchEvent<T>) => void;
type React.ClipboardEventHandler<T> = (e: React.ClipboardEvent<T>) => void;
type React.WheelEventHandler<T>     = (e: React.WheelEvent<T>) => void;
type React.PointerEventHandler<T>   = (e: React.PointerEvent<T>) => void;
```

ALWAYS use these shorthand types when typing handler props in component interfaces.
