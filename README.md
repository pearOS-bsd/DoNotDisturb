# DoNotDisturb

`/System/Library/PrivateFrameworks/DoNotDisturb.framework` (private): Focus modes (Do Not Disturb, Sleep, Personal, Work ...), assertions that turn a mode on, mode configuration and state updates. The daemon is `donotdisturbd`, reached over XPC (`com.apple.donotdisturb.service`, `com.apple.donotdisturb.service.non-launching`). Related private frameworks: `DoNotDisturbKit`, `DoNotDisturbServer`, `Focus`, `FocusEngine`, `FocusSettingsUI` (all present on the Mac).

**Status: planned. No framework code yet.** What exists today: Focus as a single on/off flag in `usernoted` (`d/bsd/usernoted`, `WSUserNotificationService`: `setFocus:` / `focusWithReply:`), used by the earlier Control Center and, as a fallback, by `apps/controlcenter-cocoa`.

## Layer
Layer 2 (client library: Foundation + XPC). The daemon is layer 4 (`d`). Used by: ControlCenter (Focus tile and page, menu-bar item), Notification Center / `usernoted` (delivery is silenced while a mode is on), System Settings (Focus pane).

## Verified API (macOS 26.7.2, probes in this repo's docs; the real ControlCenter's usage)
Everything below was run on the Mac VM from a program signed ad hoc **with the real ControlCenter's entitlements** and gave the results described.
- `+[DNDModeAssertionService serviceForClientIdentifier:]` with the client identifier **`com.apple.controlcenter.dnd`** (the real ControlCenter's `DNDClientIdentifier` in its Info.plist).
  - `-activeModeAssertionWithError:` -> `DNDModeAssertion` (`-details` -> `DNDModeAssertionDetails`: `-identifier`, `-modeIdentifier`, `-lifetime`, `-reason`, `-isUserRequested`) or nil when no mode is on.
  - `-takeModeAssertionWithDetails:error:` with `+[DNDModeAssertionDetails userRequestedAssertionDetailsWithIdentifier:modeIdentifier:lifetime:]` and `+[DNDModeAssertionLifetime lifetimeForUserRequest]` (stays on until turned off): turns the mode on. Other lifetimes seen as class methods: `lifetimeWithDuration:`, `lifetimeWithDateInterval:`, `lifetimeUntilEndOfScheduleWithIdentifier:`, `lifetimeUntilEndOfCalendarEventWithUniqueID:occurrenceDate:`, `lifetimeWithCurrentLocation`.
  - `-invalidateActiveModeAssertionWithError:` -> a `DNDModeAssertionInvalidation`: turns it off. Also `-allModeAssertionsWithError:`, `-addAssertionUpdateListener:error:` (state change callbacks), `-invalidateAllActiveModeAssertionsWithError:`.
- `+[DNDModeConfigurationService serviceForClientIdentifier:]`, `-modeConfigurationsReturningError:` -> an **NSDictionary: mode identifier -> `DNDModeConfiguration`** (`-mode` -> `DNDMode`: `-name`, `-modeIdentifier`, `-symbolImageName` (e.g. `moon.fill`), `-visibility`, `-symbolDescriptor`; `-dimsLockScreen`). On the Mac VM only `com.apple.donotdisturb.mode.default` ("Do Not Disturb") is configured.
- Mode identifiers: `com.apple.donotdisturb.mode.default` (verified), others to verify on a configured Mac: `com.apple.sleep.sleep-mode`, `com.apple.focus.personal-time`, `com.apple.focus.work`, `com.apple.donotdisturb.mode.driving`, `com.apple.focus.reading`, `com.apple.focus.gaming`, `com.apple.focus.mindfulness`, `com.apple.focus.fitness`.
- **Access control**: without the entitlements the XPC call fails with `BSServiceConnectionErrorDomain` code 3 ("XPC error received on message reply handler"). The entitlements (arrays of allowed client identifiers, here `com.apple.controlcenter.dnd` and `com.apple.focus.activity-manager`) are `com.apple.private.donotdisturb.mode.assertion.client-identifiers`, `...mode.assertion.user-requested.client-identifiers`, `...modeconfiguration.{request,modify,updates}.client-identifiers`, `...settings.{request,modify,updates}.client-identifiers`, `...state.{request,updates}.client-identifiers`, `...behavior.resolution.client-identifiers`; plus the mach-lookup exception `com.apple.donotdisturb.service` and `com.apple.private.nc.dnd.{read,write}`. Our daemon must enforce the same checks (against our code-signing / entitlement model).
- Transport: BSServiceConnection (BaseBoard), i.e. XPC with `BSProtocol` messages; the wire format is not documented yet (REA of `DoNotDisturb` and `donotdisturbd`).

## Plan
1. Daemon `donotdisturbd` in `d` (Mach service names above), state in a plist; mode set with identifiers and names as Apple's; assertions with lifetimes (user-requested lasts until invalidated; timed lifetimes expire), listeners.
2. Client classes `DNDModeAssertionService`, `DNDModeConfigurationService`, `DNDModeAssertionDetails`, `DNDModeAssertionLifetime`, `DNDMode`, `DNDModeConfiguration`, `DNDModeAssertion`, `DNDModeAssertionInvalidation` with the same selectors (verified above) so the real Control Center code and ours run unchanged.
3. `usernoted` asks the daemon whether delivery is silenced (replaces its own `focus` flag; keep `setFocus:` as a compatibility call that takes/invalidates the default-mode assertion).
4. Entitlement checks on the connection; the BS wire format from REA.

## Existing code to reuse
`d/bsd/usernoted/WSUserNotificationService.m` (focus flag), `apps/controlcenter-cocoa/CCControls.m` (the verified calls, `focusRows`).

## Rules for this repository
- Our code: BSD-2-Clause. Apple files are never copied here: reference material is read from the user's own Mac (`macdeps`), notes and measured values are written in our own words (`docs/`).
- No GPL/LGPL code. Upstream code, if any, only under `upstream/` with a `PROVENANCE.md`.
- Before writing a function: search the code index and `where-is.sh` (rule 3); do exactly what Apple does, with evidence recorded here (rule 4).
- Test on the FreeBSD VM; CI is the shared `hygiene` workflow (`pearOS-bsd/.github`).
