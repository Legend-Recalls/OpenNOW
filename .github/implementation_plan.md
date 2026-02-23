# Queue Bypass System Implementation Plan

This plan aims to implement a multi-account queue bypass system in OpenNOW, allowing the user to log in to a second account and activate a background queue while their primary account is in a session. Once the primary session ends (e.g. the 1-hour free limit) and the secondary queue is ready, the user can instantly switch over.

## User Review Required

> [!IMPORTANT]
> To implement this efficiently without major framework restructuring, we will add an `accountId` ("primary" | "secondary") parameter to the backend IPC channels and `AuthService`. The frontend will maintain a `primaryAuthSession` and a `secondaryAuthSession`.
> Is there a specific keyboard shortcut or UI location you prefer for triggering the "Start Bypass Queue" action while in-game?

## Proposed Changes

### Main Process (Backend)

#### [MODIFY] `src/main/gfn/auth.ts`
- Alter the persisted state structure from `session: AuthSession | null` to `sessions: Record<string, AuthSession>`.
- Modify `AuthService` methods (`login`, `logout`, `getSession`, etc.) to accept an `accountId: string = "primary"` parameter.
- Refactor the token refresh logic and VPC caching to be account-aware (i.e. caching by user ID or account ID).

#### [MODIFY] `src/main/gfn/cloudmatch.ts`
- Ensure that `getActiveSessions`, `createSession`, `pollSession`, and `claimSession` use the provided token exactly as passed without mutating the global state incorrectly (which they already mostly do).

#### [MODIFY] `src/shared/gfn.ts` & `src/shared/ipc.ts`
- Update IPC Request types (e.g. `AuthSessionRequest`, `AuthLoginRequest`) to add an optional `accountId?: "primary" | "secondary"` parameter.

#### [MODIFY] `src/main/index.ts`
- Pass the `accountId` argument received from frontend IPC to the `AuthService` methods.

### Preload Bridge
#### [MODIFY] `src/preload/index.ts`
- Update the definitions of `getAuthSession`, `login`, and `logout` to pass the new `accountId` structures.

### Renderer (Frontend)

#### [MODIFY] `src/renderer/src/App.tsx`
- Add `secondaryAuthSession` and `secondaryQueueStatus` state variables.
- Update the initialization logic to restore both primary and secondary sessions.
- In `handleLogin`, pass `{ accountId: 'primary' }` and create a matching `handleLoginSecondary` function.
- Create `handleStartSecondaryQueue`: fetches the active game, calls `createSession` with the secondary token, and starts a background polling loop similar to the existing `handlePlayGame` loop.
- When the secondary queue is ready, show an overlay or persistent toast "Bypass Queue Ready! [Switch]".
- Create `handleSwitchToSecondary`: stops the current primary stream and connects to the active secondary session using `claimAndConnectSession`.

#### [MODIFY] `src/renderer/src/components/SettingsPage.tsx`
- Add a new section under the main account settings to allow users to sign in and out of a "Secondary Account (Bypass Queue)".

#### [MODIFY] `src/renderer/src/components/StreamView.tsx`
- Add UI to trigger the secondary queue if a secondary account is logged in.
- Show a status indicator for the background queue (e.g. "Secondary Queue: Position 50").
- Add a prominent "Switch Session" button when the secondary queue is ready.

## Verification Plan

### Automated Tests
- Run `npm run typecheck` to ensure there are no TypeScript interface regressions.

### Manual Verification
1. Run `npm run dev`.
2. Open Settings, verify the "Secondary Account" section is visible. Sign in to a secondary GFN account.
3. Launch a game on the primary account.
4. While in the stream, click the new "Start Bypass Queue" button.
5. Watch the queue position indicator counting down in the overlay.
6. Once the queue is ready, click "Switch" and confirm the stream successfully restarts on the secondary account.
