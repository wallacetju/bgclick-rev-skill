---
name: bgclick-rev
description: Reverse-engineer a macOS GUI-automation app's background-click path (CGEvent.postToPid pattern) and produce a 1:1 Swift reproduction. Use when the user points at a closed-source macOS app that clicks background windows without activating them, and wants to understand and reproduce the mechanism. Prerequisite — IDA Pro with IDA MCP attached; skip politely if the user has no IDA.
---

# Background-Click Reverse-Engineering Skill

Audience: reverse engineers who have IDA + IDA MCP. This skill assumes Hex-Rays access and Mach-O tooling (`otool`, `nm`, `codesign`, `lldb`). Skip work and explain the blocker if IDA MCP is not attached.

## What this skill produces

A research bundle at `<workspace>/research/` plus a working Swift reproduction module and a test target. Target deliverables:

- `research/FINAL-REPORT.md` — 1:1 behaviour spec + diff vs the user's reproduction + code changes + VM test plan.
- `research/findings/Q1..QN-*.md` — one file per open question.
- `research/sub_<addr>-*.md` — one file per reverse-engineered function (rule: "every function you write you must save to `research/sub_xxx.md`").
- `research/SHARED-CONTEXT.md` — briefing doc that every sub-agent reads before working.
- `Sources/<Module>/BackgroundClicker.swift` — Swift reproduction using only public Apple APIs.
- `EchoApp/main.swift` — 500×500 red window, tap counter, bundles as `EchoApp.app` for VM regression.
- `impl-research.md` — empirical log of VM runs (windowNumber, screen point, window-local, result).

## Invariant background facts (treat as ground truth, verify against target)

The pattern these apps follow, stated in public-API terms:

1. Posting mechanism is `CGEvent.postToPid(_ pid: pid_t)` via the Swift overlay. The Mach-O will not import `_CGEventPost` / `_CGEventPostToPid` symbols; resolution goes through a dyld-walking hash resolver (salted 64-bit hashes over `LC_SYMTAB`). Do not expect `nm -u` to show CGEvent imports.
2. Event synthesis is `+[NSEvent mouseEventWithType:location:modifierFlags:timestamp:windowNumber:context:eventNumber:clickCount:pressure:]` then `-[NSEvent CGEvent]` to extract the `CGEventRef`, then explicit field writes.
3. Four explicit `CGEventSetIntegerValueField` writes, all public SDK constants:
   - `3` = `kCGMouseEventButtonNumber` ← button index (0=left, 1=right, 2=other).
   - `7` = `kCGMouseEventSubtype` ← constant `3` (non-standard; outside the public `kCGEventMouseSubtype*` enum).
   - `91` = `kCGMouseEventWindowUnderMousePointer` ← real `CGWindowID` of the target window.
   - `92` = `kCGMouseEventWindowUnderMousePointerThatCanHandleThisEvent` ← same `CGWindowID`.
4. Location pipeline:
   - public `CGEventSetLocation` with screen-space point;
   - read back via `CGEvent.location` getter;
   - apply `CGAffineTransform(translationX: -rect.origin.x, y: -rect.origin.y)` to get window-local;
   - private setter `CGEventSetWindowLocation` (resolvable via `dlsym(RTLD_DEFAULT, "CGEventSetWindowLocation")`) with the window-local point.
5. Flags trick: `event.flags = kCGEventFlagMaskCommand` (`0x00100000`) when the target app is backgrounded. This is the `⌘` modifier bit used as a WindowServer filter bypass, not `kCGEventFlagMaskNonCoalesced` (`0x00000100`) — the two constants are commonly confused. The condition is `![NSRunningApplication runningApplicationWithProcessIdentifier:pid].isActive`.
6. `CGWindowID` source: `CGWindowListCopyWindowInfo(.optionOnScreenOnly, kCGNullWindowID)` filtered by `kCGWindowOwnerPID`, then read `kCGWindowNumber`. `CGWindowListCreateDescriptionFromArray` is used for validation.
7. NSEvent.cgEvent auto-fills 12 fields: `0, 1, 2, 41, 43, 44, 50, 51, 55, 59, 102, 108`. Do not write these manually or you double-write.
8. Activation state is not touched — no `activateIgnoringOtherApps:`, no `_AXUIElementSetAttributeValue(kAXFrontmostAttribute,...)`, no SLS/CGS private APIs on the click path. If you see activation selectors in the binary, they are on unrelated UI paths.
9. "Two windows show active traffic lights simultaneously" is an AppKit side-effect of `postToPid`: the event runs the normal `sendEvent:` path, AppKit promotes the target `NSWindow` to key via `becomeKey` (if `canBecomeKey`), but the clicker's app stays frontmost. Target fires `NSWindowDidBecomeKeyNotification`; `NSApplicationDidBecomeActiveNotification` does not fire. `NSApp.isActive` stays `false` in the target.

Use these as priors when auditing. Verify each one against the target binary before relying on it — the research workflow below does exactly that.

## Plan — execute these phases in order

Treat each phase as a checkpoint. After each phase, write the artifacts listed before moving on. Use `TaskCreate` to track phases. Spawn sub-agents for any independent audit task — they must read `SHARED-CONTEXT.md` first.

### Phase 0 — Workspace + target triage (≈10 min)

1. `ls -la` the target `.app`; locate main executable and any nested `SharedSupport/*.app`. Record Mach-O paths for both service (host) and client (CLI).
2. `codesign -d --entitlements - <exe>` and `nm -u <exe> | rg '_CGEvent|_AXUIElement|_CGWindowList|_dlsym|_dlopen'` for each binary.
3. Open both Mach-Os in IDA, confirm IDA MCP on two ports (service + client). Call `ida-pro-mcp.list_instances` to verify.
4. Write `research/SHARED-CONTEXT.md` with:
   - binary paths, architectures, signing status;
   - bundle IDs;
   - IDA MCP port → binary mapping;
   - global rule: "every function you decompile, save as `research/sub_<addr>-<label>.md`";
   - "no-negation" writing rule if the user is on `CLAUDE.md` with that rule;
   - the 9 invariant facts above as "priors to verify".

### Phase 1 — CG/AX call-site audit on service binary (≈20 min, parallelisable)

Spawn one sub-agent (Explore subtype) to run, against the service IDB:

```
audit(func-level): trace UPWARD from
  _CGWindowListCopyWindowInfo, _CGWindowListCreateDescriptionFromArray,
  _kCGWindowOwnerPID, _AXUIElementPerformAction, _AXUIElementSetAttributeValue,
  _AXUIElementCopyElementAtPosition, _CGPreflightScreenCaptureAccess,
  _CGRequestScreenCaptureAccess, SCScreenshotManager.

For each hit:
  - function addr + current name
  - role: window-enum | pid-match | AX-lookup | click-dispatch | screencap | filter
  - key callers/callees (≤ 5 each)
  - whether it appears on the click path (keyword: "Dispatch click to element %ld.")

Group by subsystem. Identify the one function that logs "Dispatch click to element %ld." — that is dispatchClick.
Final output saved as research/findings/phase1-call-site-map.md.
```

Expected finding: 2–3 `dispatchClick` variants converge into a single 896-byte Swift async task frame allocator. Record its address as `SYNTH_ENTRY`.

### Phase 2 — Async continuation chain (≈20 min)

From `SYNTH_ENTRY`, walk the Swift async chain. Pattern you will see:

```
dispatchClick → allocator(896) → [6–7 continuation hops] → synthesizer
```

Each hop reads/writes into the 896-byte frame. The synthesizer is a ≈400-line Hex-Rays function containing the `+[NSEvent mouseEventWithType:...]` call and a loop over `clickCount`.

For each hop function, write `research/sub_<addr>-<label>.md` with:

- Hex-Rays listing (trimmed to ≤60 lines around the writes).
- Field-offset table: what the function writes into the frame.
- One-line Swift equivalent.

Merge all hop writes into a single **task-frame map** (`research/frame-map.md`) with every offset you observe written (at minimum: `buttonIndex`, `clickCount`, `flipY`, window rect origin/size, screen-space x/y, `isActive` byte).

### Phase 3 — dyld-hash resolver + CGEvent overlay setters (≈30 min, parallelisable)

The synthesizer calls several wrapper functions of the shape:

```c
sub_X() {
  if (qword_once != -1) swift_once(&qword_once, resolver_Y);
  return off_resolved_fnptr();
}
```

and each `resolver_Y` calls `sub_Z(&image_str_table, HASH1, HASH2, HASH3)` where `sub_Z` is a ≈440-line dyld walker (`_dyld_image_count` / `_dyld_get_image_header` / `_dyld_get_image_name`, walks `LC_SEGMENT_64` cmd=25 for `__TEXT`/`__LINKEDIT`, walks `LC_SYMTAB` cmd=2 over `nlist_64`).

Spawn one sub-agent per wrapper (up to 7) in parallel. Each sub-agent:

1. Decompile the wrapper.
2. Record the magic hash(es) and the image-hash table (`off_<addr>`).
3. Cross-check with `dlsym` on a live macOS of similar version to identify the resolved symbol.
4. Write `research/sub_<addr>-<symbolname>.md` with the verified mapping.

Expected resolver outputs (these are your priors, verify per-binary):

- `CGEventSetIntegerValueField`
- `CGEventSetLocation` (public)
- `CGEventSetWindowLocation` (private — often lives only on recent macOS)
- `CGEventGetLocation`
- `CGEventSetFlags`
- `CGEventSetTimestamp`
- `CGEventPostToPid`

Mark any wrapper whose symbol you cannot confirm as an open question.

### Phase 4 — CGEvent field map (≈15 min)

From the synthesizer, extract every `setIntegerValueField(field_id, value_expr)` call at ARM64 addresses. Cross-reference against public `CoreGraphics/CGEventTypes.h`. Build `research/findings/cgevent-fields.md` with a table:

| Field | SDK constant | Source expression | Who writes it (explicit vs NSEvent.cgEvent) |

Expected: 4 explicit writes (see invariants #3), 12 auto-filled by `NSEvent.cgEvent`. Anything outside these is new and an open question.

### Phase 5 — Open-question hunt (parallel sub-agents)

For every ambiguity from Phases 2–4, open an entry in `research/open-questions.md` (Q1, Q2, …). Typical list:

- Q1 — write site of `ctx[<offset>] = CGWindowID` (which continuation hop does it?)
- Q2 — confirm `sub_XXXX = CGEventSetWindowLocation`
- Q3–Q6 — each `setIntegerValueField` call's exact field + value + address
- Q4 — flags immediate (`0x100000` vs `0x100` confusion) + the `CSEL` condition
- Q5 — location pipeline order and which ctx offsets are screen vs window-local
- Q7 — window activation / traffic-light behaviour audit (search for activation selectors; expected answer: zero hits on click path)

Spawn one sub-agent per open question. Each agent:

1. Reads `SHARED-CONTEXT.md` and the relevant Phase 2–4 docs.
2. Uses IDA MCP (`decompile`, `xrefs_to`, `py_eval`, `imports_query`).
3. Writes `research/findings/Q<N>-<slug>.md` with: verdict in the first line, evidence addresses + Hex-Rays excerpts, implication for the reproduction.

Wait for all to complete before Phase 6.

### Phase 6 — Swift reproduction (≈30 min)

Write `Sources/<Module>/BackgroundClicker.swift` using only public Apple APIs + one `dlsym` for `CGEventSetWindowLocation`. Use the template in the Reference section below as starting point — edit only the values you verified.

Key naming rules:

- Call the flag `backgroundDispatchFlag`, assigned `CGEventFlags.maskCommand`. Never call it "nonCoalesced" — `kCGEventFlagMaskNonCoalesced` is `0x100` and is not what the target uses.
- `nextSyntheticEventNumber()` uses a process-global atomic counter or `systemUptime * 1e6 & 0x7fff_ffff`. Either works; the target uses a global counter, your reproduction can use either.
- `CGEvent.timestamp` expects nanoseconds. If the target assigns raw `systemUptime` seconds as `UInt64` without `* 1e9`, note it as a deviation — AppKit tolerates either because `NSEvent.cgEvent` overwrites timestamp on construction.

### Phase 7 — VM test harness + empirical log (≈20 min)

Create `EchoApp/main.swift` with the SwiftUI counter view (500×500, `.onTapGesture { count += 1 }`, `.windowStyle(.hiddenTitleBar)`, `.windowResizability(.contentSize)`). Template in Reference below.

Package as `EchoApp.app` with bundle id `com.demo.EchoApp`, adhoc-signed. Run in VM (TCC granted, target in background).

For each run, append to `impl-research.md`:

```
run <n>  windowNumber=<id> screen=(x,y) windowLocal=(x,y) isActive=<bool> result=<pass|fail>
```

Pass criteria: target window's counter increments, clicker app stays frontmost, target's `NSApp.isActive` stays `false`, target fires `NSWindowDidBecomeKey` only.

### Phase 8 — FINAL-REPORT

Write `research/FINAL-REPORT.md` with sections:

1. 1:1 behaviour spec of the target's click path.
2. Evidence table (each claim → verifying finding).
3. Reproduction diff — per dimension, mark `等价 / 需修正 / 存疑`.
4. Code changes list (ordered by priority).
5. VM regression test plan with specific EchoApp scenarios.
6. Unresolved low-priority questions.
7. Decision matrix — what to ship now, what to defer.

## Sub-agent discipline

- Every sub-agent reads `research/SHARED-CONTEXT.md` as its first action.
- Every sub-agent owns exactly one question or one function.
- Every sub-agent writes exactly one file named `research/findings/<slug>.md` or `research/sub_<addr>-<label>.md`.
- First line of every output file: one-sentence verdict. Evidence follows.
- Sub-agents do not modify source code. They only write research docs.

## Known failure modes

- Claiming `0x100000` is `NonCoalesced`. It is `Command`.
- Concluding `ctx[<offset>] = 0` because you did not find the write site. The write site is in a continuation hop you did not inspect. When empirical VM runs show `windowNumber != 0`, retract immediately.
- Misidentifying a Swift `Array.append(contentsOf:)` specialization as the event poster. The real poster is `CGEvent.postToPid`, resolved via `dlsym`.
- Focusing on codesign/TCC fixes. Stay on the binary audit; TCC is a VM-host concern.

## Reference — CGEvent field map (public SDK)

| Field | Constant | Filled by |
|------:|----------|-----------|
| 0 | `kCGMouseEventNumber` | `NSEvent.cgEvent` |
| 1 | `kCGMouseEventClickState` | `NSEvent.cgEvent` |
| 2 | `kCGMouseEventPressure` | `NSEvent.cgEvent` |
| 3 | `kCGMouseEventButtonNumber` | explicit |
| 7 | `kCGMouseEventSubtype` | explicit, value `3` |
| 41 | `kCGEventSourceUnixProcessID` | `NSEvent.cgEvent` |
| 43 | `kCGEventSourceUserID` | `NSEvent.cgEvent` |
| 44 | `kCGEventSourceGroupID` | `NSEvent.cgEvent` |
| 50 | (private) | `NSEvent.cgEvent` (constant `248`) |
| 51 | (private, windowNumber) | `NSEvent.cgEvent` |
| 55 | (private, event-type rawValue) | `NSEvent.cgEvent` |
| 59 | flags mirror | `NSEvent.cgEvent` |
| 91 | `kCGMouseEventWindowUnderMousePointer` | explicit, = CGWindowID |
| 92 | `kCGMouseEventWindowUnderMousePointerThatCanHandleThisEvent` | explicit, = CGWindowID |
| 102 | (private, constant `63`) | `NSEvent.cgEvent` |
| 108 | (private, button bitmap) | `NSEvent.cgEvent` |

## Reference — EchoApp test target

```swift
import SwiftUI

struct EchoApp: App {
    @State var count = 0
    var body: some Scene {
        WindowGroup {
            ZStack {
                Color.red
                Text("\(count)")
                    .foregroundStyle(.white)
                    .font(.largeTitle)
                    .contentTransition(.numericText())
            }
            .onTapGesture { count += 1 }
            .animation(.spring, value: count)
            .ignoresSafeArea()
            .frame(width: 500, height: 500, alignment: .center)
        }
        .windowStyle(.hiddenTitleBar)
        .windowResizability(.contentSize)
    }
}
EchoApp.main()
```

Package: `xcrun swiftc main.swift -o EchoApp && mkdir -p EchoApp.app/Contents/MacOS && cp EchoApp EchoApp.app/Contents/MacOS/ && codesign --force --sign - EchoApp.app`. Set `CFBundleIdentifier = com.demo.EchoApp` in `Contents/Info.plist`.

## Kickoff prompt (paste to Claude to start)

```
Use the bgclick-rev skill.
Target app: <absolute path to .app>
Workspace: <absolute path to workspace>
Execute Phase 0 now; pause for my review before each subsequent phase unless I say "go all the way".
```

## Legal & licensing

This skill contains no addresses, bytes, or decompiled code from any specific closed-source binary. It describes a methodology plus public Apple SDK constants. The Swift reproduction uses only documented AppKit / CoreGraphics APIs and one `dlsym` on a documented private symbol (`CGEventSetWindowLocation`). Fair-use reverse engineering for interoperability of your own software is the intended scope. Users of this skill are responsible for licenses of any target they analyse.
