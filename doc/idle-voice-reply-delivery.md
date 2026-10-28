# Voice reply delivery without idle polling

Phone replies wake from committed publication, reply, session, and configuration
changes. The application registers the worker with the shared delivery coordinator.
The one-second chat maintenance lane no longer calls `pushReplies`. Inbound voice
admission, report reconciliation, and other chat duties remain separate migrations.

## Durable truth and delivery boundaries

The existing voice sessions, voice replies, chat publications, and chat actions
remain authoritative. No table or migration is added. The worker reconstructs
active phone sessions and outstanding dispatch intents at startup. Notifications
are scoped hints, never permission to replay a provider action.

A reply inserted while its publication is streaming is not yet deliverable.
Both normal publication completion and operator resolution signal the endpoint
before committing its published state. Reply insertion and authorized question
retrieval signal their session before writing. The existing transaction boundary
holds notifications until outer commit and reconciles uncertain outcomes.

The dispatcher retains current authorization, session locking, ordered reply and
multipart barriers, and its atomic action claim. Provider calls remain outside
SQL transactions. An accepted receipt is distinct from spoken playback. Unknown
outcomes never authorize resending the answer or appending its remaining parts.

## Deadlines and disarming

- A ready ordered reply is immediate work; a bounded pass drains up to eight parts.
- A definitive rate limit retains its saved retry deadline.
- A dispatch without a durable receipt retains its existing strict 30-second
  uncertainty deadline. These actions are tracked independently of live sessions,
  so ending a call cannot strand its abandoned dispatch intent.
- Database or provider-path failures retain per-entry backoff. Scoped inventory
  failures retry their scope; healthy companies retain fair processing access.
- No ready reply, a presented question awaiting retrieval, accepted history, or an
  unknown/failed outcome leaves no recurring reply scan.

Live-call recovery already checks current authority before provider inspection.
Reply delivery remembers sessions denied by its authorization check. A successful
subsequent live-call authorization signals only those blocked sessions. This
preserves restoration after membership/task authority changes without adding
notifications to every authorization table or scanning healthy empty reply queues
on every live-call inspection. Startup reconstructs this in-memory state from
existing sessions. This dependency must remain explicit if active-call recovery
is changed later; a notification alone never bypasses reauthorization.

## Lifecycle and limits

The coordinator coalesces changes, serializes work, preserves dirty scopes across
failure and drain, and stops work on shutdown. Startup loads identities in pages;
processing uses fair bounded batches. Manual `pushReplies` and one-shot `reconcile`
remain available for explicit diagnostics/tests and must not become periodic lanes.

Signals cover the supported process-local database owner. Independent SQL writers
or other processes require their own notification/reconciliation contract. These
changes do not extend Cloud wake coverage, permit sleep during a live call, or
prove deployed instance/database suspension.


Both live-session recovery and reply delivery use `scoped-deadline-work.ts` for
coalescing, fair batching, scope reconstruction, retries and drain retention.
Their adapters keep domain state and provider operations in the voice owner.
The production-wiring regression covers authority returning before live recovery
observes the denial; recovery still ends calls when their authority is absent.


## Current-parent paging contract acceptance, October 27

This slice normally integrates bounded voice recovery #17576 at `712f3862b5`,
including receiver `6e88ca0201` and the landed foundations. The original 30-file
reply scope adds one shared contract path, `scoped-deadline-work.ts`. No database
file changes. The existing helper could represent only a complete inventory;
using that contract for the repaired parent would erase its bounded-pass guarantee.

The real 205-call parent regression fails before the integration repair: startup
executes three inventory queries (100, 100, 5 rows) instead of one 100-row query.
The shared helper now accepts either its existing complete array or a normalized
page result. It retains page generations and seen identities, keeps existing
entries until reconstruction completes, limits dirty scopes per pass, and replays
writes received during reconstruction. It owns no SQL or voice/provider policy.
Recovery delegates to its owner refresh and resets partial owner state on worker
registration. Existing complete-array consumers keep their public calls.
Connection observations retain the parent's separate receiver/command notification
path; actual configuration wakes both voice owners.

All 445 distinct targeted cases pass: 18 complete parent recovery lifecycle
cases, 108 complete voice-store/reply integrations, 224 complete tool-gateway and
secret-proposal cases, and 95 shared dispatcher/coordinator/batching/startup/
hostname cases. The helper's single existing test also ran in the initial
19-case green group; it is counted only in the 95-case group. The 224 cases
exercise the four other production consumers of the shared helper, including
complete-array empty/expiry/recovery/projection, transaction and drain behavior.
Server production typecheck and server build pass. The voice fixtures self-migrate
separate initially empty databases. The two other database suites use separate
migrated clones. Expected injected provider/wakeup failures and listener warnings
remain in logs. Only documentation edits follow these passing production checks.

This acceptance preserves the parent's startup paging, restart, behind-cursor
insert, concurrent feature change, health-only quiet, true disablement, current
authority, failure, drain and shutdown contracts. Existing real publication,
operator resolution, uncertain send, multipart/question ordering, authority
restoration and 105-abandoned-dispatch tests remain in the 108-case suite.
Fresh exact-head reviews and all CI remain required. No full local matrix,
live provider acceptance, Cloud wake, deployed suspension or savings is claimed.
