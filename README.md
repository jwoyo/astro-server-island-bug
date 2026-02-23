# Astro Bug: Scripts in slot content of `server:defer` components are silently dropped

## Setup

```bash
pnpm install
pnpm build
pnpm preview
```

Open http://localhost:4321

## The Bug

When an Astro component with a `<script>` tag is passed as **slot content** to a `server:defer` component, the script is silently dropped. The component's HTML renders correctly, but it has no interactivity.

### Root cause

In `server-islands.js`, non-fallback slots are serialized via:

```js
const content = await renderSlotToString(this.result, this.slots[name]);
renderedSlots[name] = content.toString(); // ← script render instructions lost here
```

`renderSlotToString` returns a `SlotString` that carries both HTML content and an `instructions` array (containing script render instructions). But `.toString()` only returns the HTML string, **silently discarding the instructions**.

The serialized slot HTML is then encrypted and sent to the server island endpoint, which recreates it via `createSlotValueFromString(content)` — a plain HTML string with no script references.

**Result**: Neither the initial page nor the server island response include the scripts from slotted components.

### Note

This is NOT the same as #11518 (fixed by `directRenderScript` in Astro 5). That issue was about scripts in **direct children** of the `server:defer` component's own template. This bug is about scripts in **slot content passed to** the component.

## Steps to Reproduce

1. `pnpm build && pnpm preview`
2. Open http://localhost:4321
3. The first counter (rendered directly) works — click "Increment"
4. The second counter (passed as slot to `server:defer`) does NOT work
5. Open DevTools Console: `✅ Counter script executed` appears only once
