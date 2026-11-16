---
title: ssrSafe
description: How to guard a store against state updates during SSR
nav: 34
---

# ssrSafe

> **Experimental:** `ssrSafe` is exported as `unstable_ssrSafe`. It is
> experimental and its name and API may change before it is finalised.
> See discussion [#2740](https://github.com/pmndrs/zustand/discussions/2740).

## Introduction

`ssrSafe` is a guard for server-side rendering. When a store is created
with it, any attempt to update state while rendering on the server throws
an error immediately, instead of silently mutating state that is shared
across requests on the server.

It does not make a store safe for SSR on its own — it only stops accidental
writes during the server render. See
[What it does not do](#what-it-does-not-do) below.

## Signature

```ts
ssrSafe<T, U>(
  config: StateCreator<T, Mps, Mcs, U>,
  isSSR?: boolean,
): StateCreator<T, Mps, Mcs, U>
```

Import it as:

```ts
import { unstable_ssrSafe } from 'zustand/middleware'
```

## Reference

### Parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `config` | `StateCreator` | Your store creator, as usual. |
| `isSSR` | `boolean` (optional) | Whether the store is rendering on the server. Defaults to `typeof window === 'undefined'`. Pass it explicitly (for example `true` in a test) to force SSR mode. |

### Behaviour

When `isSSR` is `true`:

- `set` is replaced with a function that throws
  `Error('Cannot set state of Zustand store in SSR')`.
- `api.setState` is replaced with the same throwing function.
- `get` still works, and `config` still runs — only writes are blocked.

When `isSSR` is `false` (the client), the store behaves normally.

## Usage

```ts
import { create } from 'zustand'
import { unstable_ssrSafe } from 'zustand/middleware'

type CounterState = {
  count: number
  increase: () => void
}

export const useCounterStore = create<CounterState>()(
  unstable_ssrSafe((set) => ({
    count: 0,
    increase: () => set((state) => ({ count: state.count + 1 })),
  })),
)
```

On the server, calling `increase()` — or
`useCounterStore.setState({ count: 1 })` — throws:

```
Error: Cannot set state of Zustand store in SSR
```

On the client, the same calls work normally.

To force SSR mode (for example in a test), pass the flag explicitly:

```ts
unstable_ssrSafe((set) => ({ /* ... */ }), true)
```

## What it does not do

`ssrSafe` is a guard, not a full SSR solution. It does not create a
per-request store, so a module-level store on a server is still shared
across requests. For frameworks such as Next.js App Router, you still need
to create a store per request; `ssrSafe` only stops accidental writes
during the server render.

## Troubleshooting

**`Error: Cannot set state of Zustand store in SSR`** — something called
`set` or `setState` during the server render. Move the update into an
effect, an event handler, or another client-only code path.
