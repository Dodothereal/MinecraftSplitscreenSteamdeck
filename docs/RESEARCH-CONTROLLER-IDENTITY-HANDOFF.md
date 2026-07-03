# HANDOFF: Controller identity + reconnect research → implementation

**For:** Hermes Agent (+ claude-code skill for the implementation work)
**From:** Claude Code deep-research run, 2026-07-02 (105 agents, 23 primary sources, 25 claims adversarially verified: 11 confirmed / 4 refuted / 10 unverified-by-rate-limit)
**Raw verified-claims JSON:** `docs/research-controller-identity-raw.json` (grep-able; every claim has source URL + verbatim quote + vote count)

---

## The two problems being solved

1. **Misidentification:** Inside our per-slot bwrap sandbox, Controlify shows the pad as
   **"Wireless Controller"** instead of the Steam virtual **"Microsoft X-Box 360 pad"**.
2. **Reconnect death:** We `--dev-bind` numbered nodes (`/dev/input/event13`, `js1`) into the
   sandbox. A mid-session disconnect+reconnect creates a NEW eventN on the host which is NOT
   bound inside the sandbox → that slot loses its controller permanently.

Relevant code: `minecraftSplitscreen.sh` — `find_controller_pairs()` (L166), bwrap assembly
(L200-229, binds at L221-222); `modules/controller_monitor.sh` (external↔virtual claiming,
InputPlumber D-Bus probe); `modules/instance_lifecycle.sh` (spawn_instance, `_vendor_of_js_node`,
`SDL_GAMECONTROLLER_ALLOW_STEAM_VIRTUAL_GAMEPAD=1` at L204).

---

## DIAGNOSIS (from confirmed claims — see raw JSON for quotes)

### Fact base (all 3-0 or 2-0 verified, primary sources)

- **C1. Controlify = SDL3.** Controller identity is resolved by SDL3's Linux joystick backend
  (bundled natives via the author's `sdl3java` JNA bindings), NOT by Minecraft/GLFW/evdev.
  GLFW is only a fallback when SDL natives fail to load — and the two paths name devices
  differently. [Controlify docs/architecture/controllers.mdx; Controlify README]
- **C2. Controlify reads GUID/vendor/product from SDL** (`SDL_GetGamepadGUIDForID`,
  `SDL_GetGamepadVendor/Product`) and **bundles its own gamecontrollerdb-sdl3.txt** loaded via
  `SDL_AddGamepadMappingsFromIO` — friendly names/classification depend on the device's SDL GUID
  matching that DB. A custom mapping entry can be injected the same way. [SDL3GamepadDriver.java; Controlify README]
- **C3. SDL identifies Steam virtual pads from the device node alone — no udev needed:**
  it matches `EVIOCGID` vendor/product == `28de` (Valve) / `11ff` (Steam Virtual Gamepad) and
  parses the slot number from the evdev name after `"pad "` (e.g. "Microsoft X-Box 360 pad 0"),
  and enumerates these FIRST sorted by slot. [SDL src/joystick/linux/SDL_sysjoystick.c, verified quote]
- **C4. Steam virtual pads are named "Microsoft X-Box 360 pad N"** (28de:11ff; newer SDL versions
  hardcode the name "Steam Virtual Gamepad", GUID `03006983de280000ff11000001000000` for ALL
  instances — so **VID:PID 28de:11ff is the robust per-slot identifier, not GUID or name**).
  [SDL issue #8724]
- **C5. "Wireless Controller" is the DS4's physical evdev name.** A game reporting it is seeing
  the PHYSICAL DS4 node, not Steam's virtual node. [SDL issue #8724 + kernel naming]
- **C6. Hotplug inside the sandbox:** Controlify reacts only to SDL3's
  `SDL_EVENT_JOYSTICK_ADDED/REMOVED` from `SDL_PollEvent`. In SDL's non-udev fallback path,
  hotplug = **inotify on /dev/input** (IN_CREATE|IN_DELETE|IN_MOVE|IN_ATTRIB) or a 3-second
  rescan gated on the **directory's mtime**. A reconnected pad is picked up ONLY if its new
  eventN node appears inside the sandbox's /dev/input — which static per-node binds prevent.
  [SDL_sysjoystick.c verified quote; Controlify source]

### Root-cause hypotheses (ranked)

- **H1 (primary): the wrong node is being bound.** `find_controller_pairs()` walks
  `/dev/input/js*` generically → siblings via sysfs. If it grabs the **physical DS4's** js/event
  pair (054c:09cc, name "Wireless Controller") instead of the **Steam virtual** pair (28de:11ff),
  Controlify inside the sandbox correctly reports what it was given: "Wireless Controller".
  Given C3/C5, this fully explains the symptom with no SDL/udev weirdness required.
  → **Verify first** (see Step 0 below). Note the launcher HAS `_vendor_of_js_node()` and the
  monitor's external↔virtual claiming — check whether the spawn path actually uses the virtual
  member of the pair or falls back to the raw scan.
- **H2 (secondary): SDL enumeration mode inside bwrap.** Unverified-but-likely leads (rate-limited
  before verification, treat as leads to confirm): SDL disables udev enumeration only when it
  detects a container via markers like `/.flatpak-info` or `/run/pressure-vessel`; a plain bwrap
  sandbox exposes NEITHER, so SDL may still attempt libudev enumeration — and with `/run/udev`
  absent it may enumerate nothing or degrade. If so, **creating a fake `/.flatpak-info` in the
  sandbox (bwrap `--file`) forces SDL's container path → inotify/scan fallback**, which works
  from device nodes alone (C3 shows identity needs no udev). Also unverified: udev-tag
  (`ID_INPUT_JOYSTICK`) dependence under the udev backend.
- **H3 (caveat): Controlify display-name nuance.** Verifiers REFUTED (0-3) the claim that the
  displayed name is just `SDL_GetGamepadName()` passed through — Controlify has additional
  naming logic. When testing, read Controlify's controller-name code path (multiversion/dev
  branch, `SDL3GamepadDriver`/`SDLControllerManager`) rather than assuming.

---

## RECOMMENDED ARCHITECTURE (what to implement)

### Step 0 — CONFIRM H1 (5 minutes, do this before any code)
On the host with 1 DS4 connected through Steam Input, log for each candidate node:
`udevadm info` / `EVIOCGID` vendor:product + `EVIOCGNAME` for BOTH members of what
`find_controller_pairs()` returns AND what the monitor's claiming logic pairs.
If the bound pair is 054c:* → H1 confirmed; fix the selection to bind the **28de:11ff** pair
(the virtual). Expected result: Controlify immediately shows "Microsoft X-Box 360 pad"/Xbox identity.

### Fix A — bind the RIGHT node (identity), keep current architecture
- Select per-slot nodes by **VID:PID 28de:11ff** (per C4 — not by name, not by GUID), using the
  monitor's external↔virtual claim map; never raw `js*` order.
- Belt-and-braces inside the sandbox env (already partially present):
  `SDL_GAMECONTROLLER_ALLOW_STEAM_VIRTUAL_GAMEPAD=1` (keep) and consider
  `SDL_GAMECONTROLLER_IGNORE_DEVICES=0x054c/0x09cc,0x054c/0x05c4` as defense-in-depth so even a
  leaked physical node is ignored by SDL. (SDL3 hint name: SDL_HINT_GAMECONTROLLER_IGNORE_DEVICES.)

### Fix B — reconnect-proof nodes (the structural fix)
Static binds of numbered nodes cannot survive replug (C6). Proven pattern from the research —
**per-slot persistent uinput proxy on the host**:
- Host daemon creates ONE virtual uinput device per slot at session start (stable node for the
  whole session), opens the current physical/virtual source, forwards events, and on disconnect
  keeps the virtual node alive; on reconnect re-attaches the new source to the SAME virtual node.
  The sandbox binds the proxy node once — replug becomes invisible to the game.
- Prior art validating exactly this (all in raw JSON with quotes):
  - **persistent-evdev** (github.com/aiberia/persistent-evdev) — built precisely because evdev
    consumers holding numbered nodes (QEMU) don't survive USB replug; same problem as bwrap binds.
  - **MoltenGamepad** — "Virtual gamepads are persistent across disconnects"; its system deployment
    also shows the isolation trick: udev rules chown physical pads 0600 to a dedicated user &
    strip `uaccess`, so ONLY the proxy sees hardware and games can only ever see proxy nodes.
  - **evdev-proxy** (github.com/redrampage/evdev-proxy) — minimal Rust evdev→uinput relay daemon.
  - **InputPlumber** (github.com/ShadowBlip/InputPlumber) — D-Bus input router that composites
    physical sources into virtual target pads (e.g. emulated Xbox 360). Our
    `controller_monitor.sh` ALREADY talks to InputPlumber via D-Bus — strongest candidate:
    let InputPlumber own the proxying (one composite target device per slot) instead of writing
    a custom relay.
- Give each proxy device the identity Controlify maps cleanly: ideally clone Steam-virtual
  identity (28de:11ff, name "Microsoft X-Box 360 pad N") so SDL's Steam-virtual fast path (C3)
  kicks in — or standard X360 identity (045e:028e). If a custom identity is ever needed, inject a
  gamecontrollerdb mapping line via Controlify's bundled-DB mechanism (C2).
- ALTERNATIVE (rejected): bind whole `/dev/input` + SDL ignore/allow hints for isolation. Hint
  propagation is env-only (fragile per-game), and DS4 touchpad/motion sub-devices leak. The proxy
  keeps kernel-level isolation AND fixes replug; prefer it.

### Step order for implementation
1. Step 0 diagnostic → confirm H1.
2. Fix A (bind virtual by 28de:11ff + ignore-hints) — small, ships alone, fixes identity.
3. Prototype Fix B with InputPlumber composite devices (fall back to a ~200-line python-evdev
   relay if InputPlumber's target-device API can't pin identity per slot).
4. Regression: 4-pad session; mid-game unplug/replug each pad; verify Controlify rebinds
   within its SDL_EVENT_JOYSTICK_ADDED handling and slot mapping stays stable.

## Key sources (all in raw JSON with verbatim quotes)
- Controlify: docs/architecture/controllers.mdx · SDL3GamepadDriver.java (multiversion/dev) · README
- SDL: src/joystick/linux/SDL_sysjoystick.c · issue #8724 (Steam virtual GUID/name) · issue #3889 +
  discourse 28887 (container/udev fallback — UNVERIFIED leads) · SDL3/README-linux · HINT_GAMECONTROLLER_IGNORE_DEVICES
- Valve: steam-for-linux #10175 (game sees physical despite Steam Input), #7826
- Proxies: InputPlumber · persistent-evdev · MoltenGamepad (+ its udev.rules) · evdev-proxy · kernel uinput docs
- bwrap: containers/bubblewrap #591 (joystick binding), flatpak #7, xdg-desktop-portal #536
