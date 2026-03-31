# React in the Terminal: Claude Code's Ink-Based UI System

## 1. Why React for a Terminal UI?

Opening the `src/ink/` directory in Claude Code's source tree is a double-take moment. React? In a terminal? With Yoga layout? A full DOM-style event system with capture and bubble phases? This is not your typical ncurses affair.

Claude Code's terminal UI is built on a heavily customized fork of [Ink](https://github.com/vadimdemedes/ink) -- the React renderer for command-line interfaces. But calling it "Ink" at this point is like calling a Formula 1 car a "Honda." The team has essentially rebuilt the rendering pipeline from scratch: a custom React reconciler wired to Yoga for flexbox layout, a cell-based screen buffer with interned style pools, a diff-based terminal update system, mouse tracking, text selection, hardware scroll regions, and a full DOM-style event dispatcher with capture/bubble phases. There are 346 application-level `.tsx` components in `src/components/` alone.

The payoff is enormous. React's declarative model means the entire Claude Code interface -- from the streaming markdown renderer to the diff viewer to the dialog system -- is expressed as composable components with predictable state management. You declare what the UI should look like; the framework figures out the minimal set of ANSI escape sequences to make it so. The alternative would be imperatively managing terminal cursor positions and escape codes across hundreds of features. That way lies madness.

## 2. The Rendering Pipeline

The rendering pipeline flows through four distinct phases: reconciliation, layout, painting, and terminal output. The following traces a frame from React state change to pixels on screen.

It starts in `src/ink/reconciler.ts`, which creates a custom React reconciler using `react-reconciler`:

```typescript
// src/ink/reconciler.ts:224
const reconciler = createReconciler<
  ElementNames,
  Props,
  DOMElement,
  DOMElement,
  TextNode,
  DOMElement,
  ...
>({
  getRootHostContext: () => ({ isInsideText: false }),
  // ...
})
```

This reconciler maps React elements to a lightweight DOM tree defined in `src/ink/dom.ts`. The "DOM" consists of typed nodes: `ink-root`, `ink-box`, `ink-text`, `ink-virtual-text`, `ink-link`, `ink-progress`, and `ink-raw-ansi`. Each `DOMElement` carries a `yogaNode` for layout, a `style` object, `childNodes`, event handlers, and scroll state.

When React commits a batch of updates, the reconciler's `resetAfterCommit` callback fires (`reconciler.ts:247`). This triggers `onComputeLayout` -- which runs Yoga layout synchronously -- followed by the throttled `scheduleRender`:

```typescript
// src/ink/ink.tsx:213
const deferredRender = (): void => queueMicrotask(this.onRender);
this.scheduleRender = throttle(deferredRender, FRAME_INTERVAL_MS, {
  leading: true,
  trailing: true
});
```

`FRAME_INTERVAL_MS` is 16ms (defined in `src/ink/constants.ts`), targeting 60fps. The render is deferred to a microtask so that `useLayoutEffect` hooks fire first -- critical for things like cursor positioning where one frame of lag is visible.

The `onRender` method calls the renderer (`src/ink/renderer.ts`), which walks the DOM tree and paints into a `Screen` buffer. The screen buffer is then diffed against the previous frame by `LogUpdate` (`src/ink/log-update.ts`), which produces a `Diff` -- an array of patches (cursor moves, style changes, text writes). These patches are optimized (`src/ink/optimizer.ts`) and then written to stdout as a single atomic block.

The key insight is double-buffering. The `Ink` class maintains `frontFrame` and `backFrame`:

```typescript
// src/ink/ink.tsx:196-197
this.frontFrame = emptyFrame(this.terminalRows, this.terminalColumns, ...);
this.backFrame = emptyFrame(this.terminalRows, this.terminalColumns, ...);
```

The renderer paints into the back buffer, diffs against the front buffer, and then swaps. This is exactly how GPU rendering works -- applied to terminal cells.

## 3. The Yoga Layout Engine

Yoga is Facebook's cross-platform flexbox layout engine, originally built for React Native. Claude Code uses a pure TypeScript port (`src/native-ts/yoga-layout/`) -- no WASM, no native bindings. This means layout is synchronous and available at import time.

The adapter lives in `src/ink/layout/yoga.ts`, which wraps the raw Yoga API behind a clean `LayoutNode` interface defined in `src/ink/layout/node.ts`. The interface is comprehensive -- the full flexbox model:

```typescript
// src/ink/layout/node.ts:93-152
export type LayoutNode = {
  insertChild(child: LayoutNode, index: number): void
  removeChild(child: LayoutNode): void
  calculateLayout(width?: number, height?: number): void
  setMeasureFunc(fn: LayoutMeasureFunc): void
  // ... setFlexDirection, setFlexGrow, setAlignItems, setJustifyContent,
  // setOverflow, setMargin, setPadding, setBorder, setGap, ...
}
```

The `YogaLayoutNode` adapter translates string-based layout enums (like `'flex-start'`, `'space-between'`) to Yoga's numeric constants:

```typescript
// src/ink/layout/yoga.ts:209-219
setAlignItems(align: LayoutAlign): void {
  const map: Record<LayoutAlign, Align> = {
    auto: Align.Auto,
    stretch: Align.Stretch,
    'flex-start': Align.FlexStart,
    center: Align.Center,
    'flex-end': Align.FlexEnd,
  }
  this.yoga.setAlignItems(map[align]!)
}
```

Text measurement is where things get interesting. Text nodes in the terminal have variable dimensions depending on content and container width. The `measureTextNode` function in `dom.ts` serves as Yoga's custom measure callback:

```typescript
// src/ink/dom.ts:332-374
const measureTextNode = function (
  node: DOMNode,
  width: number,
  widthMode: LayoutMeasureMode,
): { width: number; height: number } {
  const rawText =
    node.nodeName === '#text' ? node.nodeValue : squashTextNodes(node)
  const text = expandTabs(rawText)
  const dimensions = measureText(text, width)

  if (dimensions.width <= width) {
    return dimensions
  }
  // ...wrapping logic for different textWrap modes
}
```

This is called by Yoga during layout to determine how much space a text node needs. The function handles tab expansion, word wrapping, and truncation modes. When layout is in `Undefined` mode (asking for intrinsic size), pre-wrapped text with embedded newlines skips re-wrapping to prevent height inflation.

Layout runs during React's commit phase (`ink.tsx:239-258`), so `useLayoutEffect` hooks have access to fresh layout data. The Yoga counters (visited, measured, cache hits) are tracked for performance profiling.

## 4. Custom Component Library

The primitive component library in `src/ink/components/` is small but powerful. It maps directly to the DOM element types.

**Box** (`src/ink/components/Box.tsx`) is the flexbox container -- the terminal's `<div>`. It accepts the full Styles interface (flexDirection, padding, border, gap, etc.) plus event handlers:

```typescript
// src/ink/components/Box.tsx:11-46
export type Props = Except<Styles, 'textWrap'> & {
  tabIndex?: number;
  autoFocus?: boolean;
  onClick?: (event: ClickEvent) => void;
  onFocus?: (event: FocusEvent) => void;
  onBlur?: (event: FocusEvent) => void;
  onKeyDown?: (event: KeyboardEvent) => void;
  onMouseEnter?: () => void;
  onMouseLeave?: () => void;
};
```

Yes, `onClick` and `onMouseEnter` in a terminal. More on that below.

**Text** (`src/ink/components/Text.tsx`) handles styled text output with color, bold, italic, underline, strikethrough, and inverse. Bold and dim are enforced as mutually exclusive via TypeScript's discriminated union:

```typescript
// src/ink/components/Text.tsx:49-58
type WeightProps = {
  bold?: never;
  dim?: never;
} | {
  bold: boolean;
  dim?: never;
} | {
  dim: boolean;
  bold?: never;
};
```

**ScrollBox** (`src/ink/components/ScrollBox.tsx`) is perhaps the most impressive primitive. It is a `Box` with `overflow: scroll` and a full imperative scroll API exposed via `useImperativeHandle`:

```typescript
// src/ink/components/ScrollBox.tsx:10-62
export type ScrollBoxHandle = {
  scrollTo: (y: number) => void;
  scrollBy: (dy: number) => void;
  scrollToElement: (el: DOMElement, offset?: number) => void;
  scrollToBottom: () => void;
  getScrollTop: () => number;
  getScrollHeight: () => number;
  getViewportHeight: () => number;
  isSticky: () => boolean;
  subscribe: (listener: () => void) => () => void;
  setClampBounds: (min: number | undefined, max: number | undefined) => void;
};
```

The scroll implementation bypasses React entirely for performance. `scrollTo` and `scrollBy` mutate the DOM node's `scrollTop` directly, mark it dirty, and schedule an Ink render via microtask. This means scroll events do not trigger React reconciliation -- they go straight to the render pipeline:

```typescript
// src/ink/components/ScrollBox.tsx:103-117
function scrollMutated(el: DOMElement): void {
  markScrollActivity();
  markDirty(el);
  markCommitStart();
  notify();
  if (renderQueuedRef.current) return;
  renderQueuedRef.current = true;
  queueMicrotask(() => {
    renderQueuedRef.current = false;
    scheduleRenderFrom(el);
  });
}
```

The renderer supports viewport culling -- only children intersecting the visible scroll window are painted. There is also a `pendingScrollDelta` mechanism that drains scroll at a maximum rate per frame, so fast scroll flicks show intermediate frames instead of jumping.

**Button** (`src/ink/components/Button.tsx`) provides interactive state via a render prop: `focused`, `hovered`, and `active`. It handles Enter, Space, and click activation, with a visual press feedback timer. It is intentionally unstyled -- the render prop gives consumers full control.

**AlternateScreen** (`src/ink/components/AlternateScreen.tsx`) switches the terminal to the alternate screen buffer (DEC private mode 1049), constrains the layout to the viewport height, and optionally enables SGR mouse tracking. This is what turns Claude Code from a scrolling-text CLI into a full-screen application.

Beyond the primitives, the `src/components/design-system/` directory contains higher-level building blocks: `Dialog`, `Tabs`, `FuzzyPicker`, `ProgressBar`, `Divider`, `Pane`, `ThemedBox`, `ThemedText`, and a full theming system via `ThemeProvider`.

## 5. Event System

The event system in `src/ink/events/` is a faithful port of the browser's DOM event model. That is literal -- it implements capture and bubble phases with propagation control.

`TerminalEvent` (`src/ink/events/terminal-event.ts`) is the base class, mirroring `Event`:

```typescript
// src/ink/events/terminal-event.ts:19-37
export class TerminalEvent extends Event {
  readonly type: string
  readonly timeStamp: number
  readonly bubbles: boolean
  readonly cancelable: boolean
  // target, currentTarget, eventPhase, stopPropagation(), preventDefault()
}
```

The `Dispatcher` (`src/ink/events/dispatcher.ts`) implements the two-phase accumulation pattern from react-dom. When an event fires, it walks from the target to the root, collecting handlers:

```typescript
// src/ink/events/dispatcher.ts:46-79
function collectListeners(target, event): DispatchListener[] {
  const listeners: DispatchListener[] = []
  let node = target
  while (node) {
    const captureHandler = getHandler(node, event.type, true)
    const bubbleHandler = getHandler(node, event.type, false)
    if (captureHandler) {
      listeners.unshift({ node, handler: captureHandler, phase: 'capturing' })
    }
    if (bubbleHandler && (event.bubbles || isTarget)) {
      listeners.push({ node, handler: bubbleHandler, phase: 'bubbling' })
    }
    node = node.parentNode
  }
  return listeners
}
```

The result is `[root-cap, ..., parent-cap, target-cap, target-bub, parent-bub, ..., root-bub]` -- exactly like a browser.

Event priorities are mapped to React's scheduling priorities (`src/ink/events/dispatcher.ts:122-138`): keyboard, click, focus, and paste get `DiscreteEventPriority` (synchronous); resize, scroll, and mousemove get `ContinuousEventPriority`. The dispatcher integrates with the reconciler so that state updates inside event handlers are batched correctly:

```typescript
// src/ink/events/dispatcher.ts:207-218
dispatchDiscrete(target, event): boolean {
  if (!this.discreteUpdates) {
    return this.dispatch(target, event)
  }
  return this.discreteUpdates(
    (t, e) => this.dispatch(t, e), target, event, undefined, undefined,
  )
}
```

`KeyboardEvent` (`src/ink/events/keyboard-event.ts`) normalizes raw terminal escape sequences into a browser-like API with `key`, `ctrl`, `shift`, `meta`, and `fn` modifiers. The handler prop registry (`src/ink/events/event-handlers.ts`) maps event types to handler names: `onKeyDown`, `onKeyDownCapture`, `onFocus`, `onBlur`, `onClick`, `onMouseEnter`, `onMouseLeave`.

Mouse support deserves special mention. When `AlternateScreen` enables SGR mouse tracking, terminal mouse events are parsed and dispatched through a hit-testing system (`src/ink/hit-test.ts`). The hit test walks the DOM tree using cached screen-space rectangles, traversing children in reverse order (so later siblings, painted on top, win):

```typescript
// src/ink/hit-test.ts:18-41
export function hitTest(node, col, row): DOMElement | null {
  const rect = nodeCache.get(node)
  if (!rect) return null
  if (col < rect.x || col >= rect.x + rect.width ||
      row < rect.y || row >= rect.y + rect.height) {
    return null
  }
  for (let i = node.childNodes.length - 1; i >= 0; i--) {
    const child = node.childNodes[i]!
    if (child.nodeName === '#text') continue
    const hit = hitTest(child, col, row)
    if (hit) return hit
  }
  return node
}
```

Focus management (`src/ink/focus.ts`) is DOM-like: a `FocusManager` tracks `activeElement`, maintains a focus stack (max 32 entries), and dispatches focus/blur events through the event dispatcher. Tab cycling, auto-focus, click-to-focus, and focus restoration on node removal are all supported.

## 6. Application Components

The 346 `.tsx` files in `src/components/` form the actual Claude Code UI. The composition hierarchy starts from `src/components/App.tsx`, which wraps providers for FPS metrics, stats, and app state using React Compiler output:

```typescript
// src/components/App.tsx:19-55 (compiled output)
export function App(t0) {
  const $ = _c(9);
  const { getFpsMetrics, stats, initialState, children } = t0;
  // ...memoization cache checks via _c()...
  return <FpsMetricsProvider><StatsProvider><AppStateProvider>
    {children}
  </AppStateProvider></StatsProvider></FpsMetricsProvider>;
}
```

The application tree includes:

- **Message rendering**: `Messages.tsx`, `MessageRow.tsx`, `MessageModel.tsx`, `MessageResponse.tsx` -- the streaming conversation display
- **Virtual scrolling**: `VirtualMessageList.tsx` -- a virtualized list that only mounts messages visible in the viewport, with height caching and search integration
- **Input**: `PromptInput/`, `TextInput.tsx`, `VimTextInput.tsx`, `BaseTextInput.tsx` -- a full text input system with Vim bindings support
- **Diff viewing**: `StructuredDiff/`, `FileEditToolDiff.tsx` -- syntax-highlighted, structured diff display
- **Code highlighting**: `HighlightedCode/` -- terminal-aware syntax highlighting
- **Markdown rendering**: `Markdown.tsx`, `MarkdownTable.tsx` -- full markdown rendering in the terminal
- **Dialogs**: at least 20 dialog components for various flows (OAuth, MCP server approval, settings, onboarding, etc.)
- **Tool UI**: tool-specific message rendering for file edits, shell commands, MCP tools
- **Navigation**: `GlobalSearchDialog.tsx`, `HistorySearchDialog.tsx`, `QuickOpenDialog.tsx`
- **Design system**: `design-system/Dialog.tsx`, `Tabs.tsx`, `FuzzyPicker.tsx`, `ProgressBar.tsx`, `ThemedBox.tsx`, `ThemedText.tsx`

The `FullscreenLayout.tsx` component orchestrates the main fullscreen view: a scroll chrome context, the message list, and the input area, all composed within `AlternateScreen`.

## 7. Hooks

The hooks in `src/ink/hooks/` provide the ergonomic API for terminal interactions.

**useInput** (`src/ink/hooks/use-input.ts`) is the keyboard handler. It subscribes to stdin events and provides a clean `(input, key, event)` callback. Raw mode is enabled via `useLayoutEffect` (not `useEffect`) to avoid a flash of cooked-mode terminal between mount and effect:

```typescript
// src/ink/hooks/use-input.ts:50-60
useLayoutEffect(() => {
  if (options.isActive === false) return
  setRawMode(true)
  return () => { setRawMode(false) }
}, [options.isActive, setRawMode])
```

**useAnimationFrame** (`src/ink/hooks/use-animation-frame.ts`) provides synchronized animations that pause when offscreen. It uses a shared `ClockContext` so all animations tick together, and `useTerminalViewport` for visibility detection:

```typescript
// src/ink/hooks/use-animation-frame.ts:30-57
export function useAnimationFrame(intervalMs: number | null = 16) {
  const clock = useContext(ClockContext)
  const [viewportRef, { isVisible }] = useTerminalViewport()
  const [time, setTime] = useState(() => clock?.now() ?? 0)
  const active = isVisible && intervalMs !== null
  // ...subscribes to clock with keepAlive when visible
  return [viewportRef, time]
}
```

The clock automatically slows when the terminal is blurred. Pass `null` to pause (unsubscribe). This powers spinners, progress indicators, and other animated elements -- all running at 60fps when visible, sleeping when off-screen.

**useSelection** (`src/ink/hooks/use-selection.ts`) exposes the text selection system: copy, clear, drag, shift-extend, word/line selection boundaries. It reads the Ink instance's selection state via `useSyncExternalStore` for reactive updates.

**useTerminalViewport** detects whether an element is within the visible scroll region. **useDeclaredCursor** lets a component declare where the terminal's native cursor should be positioned (critical for CJK IME input). **useTerminalFocus** tracks whether the terminal window has focus (via DEC focus event reporting).

## 8. Streaming Output

Model responses in Claude Code stream token-by-token from the API. The React rendering model handles this naturally: as tokens arrive, message state updates, React re-renders the message component, Yoga recomputes layout, and the renderer diffs the screen buffer.

The key optimization is the dirty-tracking system in the DOM. When text content changes, `markDirty` walks from the changed node up to the root, setting `dirty = true` on each ancestor:

```typescript
// src/ink/dom.ts:393-413
export const markDirty = (node?: DOMNode): void => {
  let current = node
  let markedYoga = false
  while (current) {
    if (current.nodeName !== '#text') {
      (current as DOMElement).dirty = true
      if (!markedYoga &&
          (current.nodeName === 'ink-text' || current.nodeName === 'ink-raw-ansi') &&
          current.yogaNode) {
        current.yogaNode.markDirty()
        markedYoga = true
      }
    }
    current = current.parentNode
  }
}
```

The renderer uses these dirty flags for blit optimization: unchanged subtrees are copied directly from the previous frame's screen buffer (via `blitRegion` in `screen.ts`) without re-rendering. During streaming, only the actively-changing message and its ancestors are re-rendered; the rest of the conversation is blitted. This is the "narrow damage" path.

The `ScrollBox` component's `stickyScroll` mode auto-pins to the bottom as content grows -- essential for a streaming conversation view. The renderer detects when scrollTop was at the previous maximum and content grew, automatically advancing the scroll position ("positional follow").

The `ink-raw-ansi` node type is an optimization for pre-rendered content. Components like `ColorDiff` produce ANSI strings with known dimensions -- they skip the text measurement, wrapping, and style interning pipeline entirely:

```typescript
// src/ink/dom.ts:379-387
const measureRawAnsiNode = function (node: DOMElement): {
  width: number; height: number
} {
  return {
    width: node.attributes['rawWidth'] as number,
    height: node.attributes['rawHeight'] as number,
  }
}
```

## 9. Performance: React Compiler and Render Optimization

Claude Code uses the React Compiler. Every compiled component opens with the characteristic cache array:

```typescript
// src/ink/components/Box.tsx:1
import { c as _c } from "react/compiler-runtime";
// ...
function Box(t0) {
  const $ = _c(42);
  // ...42 memoization slots
```

The compiler automatically memoizes intermediate values and JSX elements, replacing manual `useMemo`/`useCallback` calls. The `Button` component has 30 cache slots; `AlternateScreen` has 7. This is critical in a terminal renderer where every unnecessary re-render means more ANSI escape sequences written to stdout.

Beyond the compiler, the rendering pipeline has several layers of optimization:

1. **Style interning**: The `StylePool` (`src/ink/screen.ts`) assigns numeric IDs to unique style combinations. The `CharPool` interns character strings. Hyperlinks get their own pool. This means screen buffer cells are just integers -- comparison is `===` instead of deep object equality.

2. **Diff optimization**: The `optimize()` function (`src/ink/optimizer.ts`) makes a single pass over the diff to remove empty patches, merge consecutive cursor moves, collapse adjacent style transitions, deduplicate hyperlinks, and cancel cursor hide/show pairs.

3. **Blit path**: When a subtree's dirty flag is false, `renderNodeToOutput` copies its cells directly from the previous frame's screen buffer. The comment in `renderer.ts` calls this the "O(unchanged) fast path for steady-state frames."

4. **Scroll drain**: Fast scroll flicks accumulate in `pendingScrollDelta` and drain at `SCROLL_MAX_PER_FRAME` rows per frame, showing intermediate frames instead of one big jump.

5. **Hardware scroll regions**: When a `ScrollBox`'s scrollTop changes between frames and nothing else moves, the renderer emits DECSTBM (set scroll region) + SU/SD (scroll up/down) instead of rewriting the viewport. This is the `ScrollHint` mechanism in `render-node-to-output.ts`.

6. **Pool reset**: Style and hyperlink pools accumulate entries over time. The Ink instance periodically resets them (every 5 minutes for hyperlinks) to prevent unbounded growth, migrating live screen data to fresh pools.

Performance instrumentation is built in: `CLAUDE_CODE_COMMIT_LOG` enables per-commit timing, `CLAUDE_CODE_DEBUG_REPAINTS` traces which components cause full-screen repaints, and Yoga counters track layout cache hit rates.

