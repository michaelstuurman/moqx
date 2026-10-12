## moqx plugins

---

## Levels

| Level | Plugin is | For |
| --- | --- | --- |
| 1 | C++ compiled into moqx | In-tree plugins and forks that need native speed and accept a rebuild |
| 2 | A shared library, loaded at startup | Plugins shipped as separate packages |
| 3 | A trusted WebAssembly module | Operators changing behavior without a restart |
| 4 | An untrusted WebAssembly module | Tenants running their own code at the edge |

---

## Level 1

- Plugins are C++ and use folly directly.
- They live in the moqx tree, or a fork adds them through its CMake build.
- Adding a plugin means rebuilding moqx.
- The prototype already builds Auth, RelayHops and the cache as plugins.

---

## Level 1: boot

1. Config lists plugins by type name.
2. One list in the code maps each type name to a factory.
3. Each factory creates a `Plugin` per config entry.
4. Each `Plugin` creates a `ServicePlugin` per service it applies to.
5. Each `ServicePlugin` creates its worker instances on every I/O worker.
6. Each `ServicePlugin` starts, in config order.
7. The service accepts sessions.

---

## Control plane vs data plane

- Control plane hooks run once per session, request or announcement.
- Admitting a session is a control plane hook.
- Data plane hooks run per object.
- Filtering an object is a data plane hook.

---

## Level 1: hooks

![Hook points](hooks.svg)

---

## Coroutines vs callbacks

- The relay's control plane runs on coroutines.
- The relay's data plane already avoids coroutines.
  - A filter that needs to wait gets a callback.

---

## Filters: rules

- A filter may observe, delay or drop objects.
- A filter never modifies an object's content.
- A filter may modify an object's properties.

---

## Filters: configured vs attached

- Configured filters are chosen once, when a subscribe, publish or fetch starts.
- A request update keeps them.
- Attached filters are added and removed at runtime.
- The control plane attaches them, per request or per track.

---

## Use case: watermark one subscriber

```text
subscriber ── SUBSCRIBE live/video/wm-<id> ──▶ relay
plugin     ── SUBSCRIBE live/video ──────────▶ relay   (local session)
plugin     ── PUBLISH   live/video/wm-<id> ──▶ relay   (new objects)
```

- A local session is an in-process peer that publishes and subscribes.
- Each subscriber subscribes to a track unique to it.
- Authorize ties `wm-<id>` to that subscriber.
- The plugin publishes the unique track, with synthesized objects.
- The source track's objects stay untouched.

---

## Use case: filter a track on demand

- An operator asks to stop delivering one track, through an admin route.
- The plugin attaches a dropping filter to every subscription of that track, live and new.
- Detaching stops it from each subscription's next stream on.

---

## Use case: relay hops

```text
relay-a ──▶ relay-b ──▶ relay-c ──▶ relay-a   (dropped)
```

- Over `RELAY_HOPS` links, an announcement carries the Hop IDs of the relays it crossed.
- The plugin drops an announcement that already holds its own Hop ID.
- The plugin appends its own Hop ID when forwarding.
- Both are announcement policy hooks.

---

## Use case: default upstream

- A subscribe with no local publisher goes to the upstream relay.
- Today the upstream is core config, not a plugin.
- As a plugin, it needs a routing hook.
- That hook does not exist yet.

---

## Use case: watermark + default upstream

```text
subscriber ── SUBSCRIBE live/video/wm-<id> ──▶ edge
watermark  ── SUBSCRIBE live/video ──────────▶ edge ──▶ upstream
watermark  ── PUBLISH   live/video/wm-<id> ──▶ edge
```

- The watermark plugin's local session subscribes like any other peer.
- The default upstream forwards that subscribe.
- The upstream carries one copy of the source, whatever the subscriber count.
- The `wm-<id>` subscribe must stay at the edge.
- It arrives before the watermark plugin publishes `wm-<id>`.
- Without a hold, the default upstream sends it upstream.

---

## How we start

- Start with Level 1.
- Land the Level 1 API with Auth, RelayHops and the cache as its first plugins.

---

## Open questions

- Should a request update re-run authorization and filter selection?
- How does a plugin learn that a track like `wm-<id>` was requested?
- Which plugin decides where a request with no local publisher goes?
- How do plugins expose metrics and admin routes?
- When do we need Level 2?
- Which plugin beyond Auth, RelayHops and the cache proves the API?
