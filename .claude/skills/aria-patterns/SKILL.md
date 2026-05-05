---
name: aria-patterns
description: >
  ARIA widget patterns for complex interactive components. Covers ARIA tree (role="tree",
  roving tabindex, keyboard navigation), ARIA grid (role="grid", virtual row aria-rowindex),
  focus management, prefers-reduced-motion, and axe-core test assertions. Activate this skill
  when implementing keyboard-navigable widgets, accessibility audits, or writing tests that
  assert ARIA attributes.
---

# ARIA Widget Patterns

Opinionated, production-tested patterns for complex interactive widgets — ARIA tree, ARIA grid, roving tabindex, focus management, motion preferences, and accessibility testing.

---

## When to Use This Skill

Apply these patterns whenever you implement a widget that is **not a native HTML element** but behaves like one:

| Widget | ARIA role | Native equivalent |
|--------|-----------|-------------------|
| Playlist / file tree | `tree` + `treeitem` | No native equivalent |
| Virtualized data table | `grid` + `row` + `gridcell` | `<table>` (unusable with virtualization) |
| Dropdown list | `listbox` + `option` | `<select>` |
| Autocomplete | `combobox` + `listbox` | `<input>` + `<datalist>` |

If a native element can do the job, use it — no ARIA needed. Apply this skill only for the cases above where native elements fall short.

---

## ARIA Tree Widget

### Role Structure

```tsx
<ul role="tree" aria-label="Playlists">
  <li
    role="treeitem"
    aria-expanded={isOpen}   // folders only — omit on leaves
    aria-selected={isActive}
    tabIndex={isFocused ? 0 : -1}
  >
    <span>My Folder</span>
    <ul role="group">        // children of a folder
      <li role="treeitem" aria-selected={false} tabIndex={-1}>
        <span>Playlist A</span>
      </li>
    </ul>
  </li>
</ul>
```

**Rules:**
- `role="tree"` on the outermost container only.
- `role="group"` on a folder's child list — never a second `role="tree"`.
- `aria-expanded` belongs on folder items (`type === 'folder'`). Leaf items do not get `aria-expanded` at all (not even `false`).
- `aria-selected` on every item — `true` for the active item, `false` for all others.
- Only one item has `tabIndex={0}` at a time (roving tabindex). All others are `tabIndex={-1}`.

### Keyboard Algorithm

The keyboard handler operates on a **flat list of currently visible items** — rebuild this list whenever expand/collapse state changes:

```tsx
function buildVisibleItems(
  nodes: PlaylistNode[],
  expandedIds: Set<string>,
  result: string[] = [],
): string[] {
  for (const node of nodes) {
    result.push(node.id);
    if (node.type === 'folder' && expandedIds.has(node.id)) {
      buildVisibleItems(node.children, expandedIds, result);
    }
  }
  return result;
}
```

Key bindings per the [ARIA Authoring Practices tree pattern](https://www.w3.org/WAI/ARIA/apg/patterns/treeview/):

| Key | Action |
|-----|--------|
| `ArrowDown` | Move focus to next visible item |
| `ArrowUp` | Move focus to previous visible item |
| `ArrowRight` | If folder and collapsed → expand. If folder and expanded → move to first child. If leaf → no-op. |
| `ArrowLeft` | If folder and expanded → collapse. If item is a child → move focus to parent. |
| `Enter` / `Space` | Activate (select) the focused item |
| `Home` | Move focus to first item |
| `End` | Move focus to last visible item |

```tsx
function handleKeyDown(e: React.KeyboardEvent) {
  const visible = buildVisibleItems(nodes, expandedIds);
  const currentIndex = visible.indexOf(focusedId);

  switch (e.key) {
    case 'ArrowDown':
      e.preventDefault();
      setFocusedId(visible[Math.min(currentIndex + 1, visible.length - 1)]);
      break;
    case 'ArrowUp':
      e.preventDefault();
      setFocusedId(visible[Math.max(currentIndex - 1, 0)]);
      break;
    case 'ArrowRight': {
      const node = findNode(focusedId);
      if (node?.type === 'folder' && !expandedIds.has(focusedId)) {
        setExpandedIds(prev => new Set([...prev, focusedId]));
      } else if (node?.type === 'folder') {
        setFocusedId(node.children[0]?.id ?? focusedId);
      }
      break;
    }
    case 'ArrowLeft': {
      const node = findNode(focusedId);
      if (node?.type === 'folder' && expandedIds.has(focusedId)) {
        setExpandedIds(prev => { const s = new Set(prev); s.delete(focusedId); return s; });
      } else {
        const parent = findParent(focusedId);
        if (parent) setFocusedId(parent.id);
      }
      break;
    }
    case 'Enter':
    case ' ':
      e.preventDefault();
      onSelect(focusedId);
      break;
  }
}
```

---

## ARIA Grid Widget

Use `role="grid"` for non-native tables, especially when combined with virtualization (`@tanstack/react-virtual`). A `<table>` is unusable with a virtualizer because virtualizers render only a window of rows, which breaks table layout.

### Role Structure

```tsx
<div
  role="grid"
  aria-rowcount={totalRows}   // total rows including non-rendered virtual ones
  aria-label="Track list"
>
  {/* Sticky header row */}
  <div role="row">
    <div role="columnheader" aria-sort="ascending">Title</div>
    <div role="columnheader" aria-sort="none">BPM</div>
  </div>

  {/* Virtualizer container */}
  <div style={{ height: totalHeight, position: 'relative' }}>
    {virtualRows.map(vr => (
      <div
        key={vr.index}
        role="row"
        aria-rowindex={vr.index + 2}  // 1-based; +2 accounts for header row
        aria-selected={selectedIndex === vr.index}
        style={{ position: 'absolute', top: vr.start }}
      >
        <div role="gridcell">{tracks[vr.index].name}</div>
        <div role="gridcell">{tracks[vr.index].bpm}</div>
      </div>
    ))}
  </div>
</div>
```

**Rules:**
- `aria-rowcount` on the `role="grid"` element = total row count including all virtual rows.
- `aria-rowindex` on each `role="row"` = 1-based position in the full dataset. The header row is index 1; data rows start at 2.
- `aria-sort` on `role="columnheader"`: `"ascending"` | `"descending"` | `"none"` | `"other"`. Only the active sort column gets `"ascending"` or `"descending"`; others get `"none"`.
- `aria-selected` on each data row — `true` for the focused/selected row.

### Grid Keyboard Algorithm

| Key | Action |
|-----|--------|
| `ArrowDown` | Move focus to row + 1; scroll into view |
| `ArrowUp` | Move focus to row - 1; scroll into view |
| `PageDown` | Move focus + 10 rows; scroll into view |
| `PageUp` | Move focus - 10 rows; scroll into view |
| `Home` | Move focus to row 0; scroll to top |
| `End` | Move focus to last row; scroll to bottom |

With `@tanstack/react-virtual`, use `virtualizer.scrollToIndex(index)` for all scroll-into-view calls.

---

## Roving Tabindex Pattern

Only **one element** in a composite widget (`tree`, `grid`, `listbox`) should be in the natural tab sequence (`tabIndex={0}`) at a time. All others use `tabIndex={-1}` so Tab skips the widget and jumps to the next focusable region.

```tsx
// In parent — track which item has the roving focus
const [focusedId, setFocusedId] = useState<string>(items[0].id);

// In each item
<li
  tabIndex={id === focusedId ? 0 : -1}
  ref={id === focusedId ? activeFocusRef : undefined}
  onFocus={() => setFocusedId(id)}
>
```

When `focusedId` changes programmatically (e.g., from a keyboard event), imperatively focus the new element:

```tsx
const activeFocusRef = useRef<HTMLLIElement>(null);

useEffect(() => {
  activeFocusRef.current?.focus({ preventScroll: true });
}, [focusedId]);
```

`preventScroll: true` is important — let your own scroll logic handle viewport position rather than the browser's default scroll.

---

## Focus Management

### Programmatic focus via refs

```tsx
const itemRefs = useRef<Map<string, HTMLElement>>(new Map());

// Register
<li ref={el => { if (el) itemRefs.current.set(id, el); else itemRefs.current.delete(id); }}>

// Focus programmatically
itemRefs.current.get(targetId)?.focus({ preventScroll: true });
```

### Scroll into view after keyboard nav

After moving focus to a virtual row, call both the virtualizer scroll and the DOM focus:

```tsx
virtualizer.scrollToIndex(newIndex, { align: 'auto' });
// Allow the virtualizer to render the row before focusing
requestAnimationFrame(() => {
  rowRefs.current.get(newIndex)?.focus({ preventScroll: true });
});
```

### Restoring focus on overlay/modal close

Always restore focus to the trigger element when a modal or popover closes:

```tsx
const triggerRef = useRef<HTMLButtonElement>(null);

function closeModal() {
  setIsOpen(false);
  // Defer to next tick so the modal unmounts first
  setTimeout(() => triggerRef.current?.focus(), 0);
}
```

---

## `prefers-reduced-motion`

Never animate elements unconditionally. Always respect the user's OS motion preference.

### CSS approach (preferred)

```css
.item {
  transition: transform 0.15s ease;
}

@media (prefers-reduced-motion: reduce) {
  .item {
    transition: none;
  }
}

/* For transform animations (scale, rotate) */
.chevron {
  transform: rotate(0deg);
  transition: transform 0.2s ease;
}

.chevron--expanded {
  transform: rotate(90deg);
}

@media (prefers-reduced-motion: reduce) {
  .chevron,
  .chevron--expanded {
    transition: none;
  }
}
```

### React inline approach (when CSS modules aren't available)

```tsx
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

<div
  style={{
    transform: isExpanded ? 'rotate(90deg)' : 'rotate(0deg)',
    transition: prefersReducedMotion ? 'none' : 'transform 0.2s ease',
  }}
/>
```

Cache the `matchMedia` result outside the component — it is synchronous and does not change per render.

---

## Testing ARIA Widgets

### Query by role — always prefer semantic queries

```tsx
// ✅ Query by role — matches what screen readers see
screen.getByRole('tree')
screen.getByRole('treeitem', { name: 'My Playlist' })
screen.getByRole('gridcell', { name: '128 BPM' })
screen.getByRole('columnheader', { name: 'BPM' })

// ✅ Check aria attributes
expect(screen.getByRole('treeitem', { name: 'My Folder' })).toHaveAttribute('aria-expanded', 'false');
expect(screen.getByRole('row', { name: /track one/i })).toHaveAttribute('aria-selected', 'true');
```

### Keyboard navigation with `userEvent`

```tsx
import userEvent from '@testing-library/user-event';

it('should move focus to next item on ArrowDown', async () => {
  const user = userEvent.setup();
  render(<PlaylistTree nodes={fixture} />);

  const firstItem = screen.getByRole('treeitem', { name: 'Folder A' });
  firstItem.focus();

  await user.keyboard('{ArrowDown}');

  expect(screen.getByRole('treeitem', { name: 'Playlist B' })).toHaveFocus();
});

it('should expand folder on ArrowRight', async () => {
  const user = userEvent.setup();
  render(<PlaylistTree nodes={fixture} />);

  screen.getByRole('treeitem', { name: 'Folder A' }).focus();
  await user.keyboard('{ArrowRight}');

  expect(screen.getByRole('treeitem', { name: 'Folder A' }))
    .toHaveAttribute('aria-expanded', 'true');
});
```

### `@axe-core/react` integration

Install as a dev dependency: `npm install --save-dev @axe-core/react axe-core`.

Add an axe assertion to every major component test suite:

```tsx
import { axe, toHaveNoViolations } from 'jest-axe';
// For Vitest: import { configureAxe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

it('should have no accessibility violations', async () => {
  const { container } = render(<PlaylistTree nodes={fixture} />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

Run axe assertions on the **rendered, fully interactive state** — test each significant visual state (collapsed tree, expanded tree, selected item) separately since different states may have different violations.

---

## Common Pitfalls

**`aria-expanded` on leaf nodes**
Leaf nodes (playlists, files) must not have `aria-expanded` at all. Setting `aria-expanded={false}` on a leaf tells assistive technology the item is a collapsible folder — it is not. Omit the attribute entirely.

**`aria-rowindex` drift in virtual grids**
If rows are rendered in a non-sequential window, `aria-rowindex` must reflect the item's position in the full dataset — not its position in the render window. A row at dataset index 500 gets `aria-rowindex={502}` (1-based + 1 for header), regardless of where it sits in the DOM.

**Roving tabindex and natural Tab order**
When the user presses Tab inside a tree/grid, focus should leave the widget entirely and move to the next focusable region. Ensure no item inside the widget has a permanent `tabIndex={0}` — only the currently active item gets `0`; the rest are `−1`.

**Keyboard handler missing `e.preventDefault()`**
Arrow keys scroll the page by default. Grid `PageDown/Up` scrolls the page by a full viewport. Always call `e.preventDefault()` inside any keyboard handler that processes arrow, Page, Home, or End keys — otherwise both the widget and the browser scroll simultaneously.

**Focusing a virtual row before it is rendered**
`virtualizer.scrollToIndex()` is asynchronous — the target row may not be in the DOM yet when you call `.focus()`. Wrap the focus call in `requestAnimationFrame()` to let the virtualizer commit the new render first.
