# Floor Plan App V2 — Setup Notes

Running record of config values created during AWS/Google setup. Reference this when writing frontend/Lambda code.

## Phase 2b - per-opening conflict resolution (see phase2b-conflict-resolution-notes.md for the
original design decisions - Nathan brought that file in from a separate session)

### Round 9a - wall auto-lock in crew view - BUILT
Crew view now always behaves as if walls are locked (`wallsEffectivelyLocked()`), regardless of
the level's real `locks.walls` flag - never mutates that flag, so a manager's own lock choice for
full view is always preserved exactly as they left it. Only walls are covered - moving/resizing
existing openings is still fully available to crew, unchanged. No AWS changes - frontend only.

### Round 9b - Lambda: openings/labels move to per-key maps, per-key merge - BUILT, NEEDS DEPLOY
This is the bigger structural piece of Phase 2b. index.mjs's savePlan() is substantially
rewritten. What changed and why:

- Two separate conflict mechanisms now instead of one:
  1. **Structural** (levels list, each level's vertices/locks/name/interiorLines, plus
     top-level fields like title/columnVisibility) - still a single whole-plan optimistic-lock
     gate, same idea as before, but scoped to a new `structuralLastEditedAt` field instead of
     the old `lastEditedAt` - so someone else's pure opening/label edit doesn't falsely trip a
     structural conflict for you (this was the key subtlety: if it shared the old field, two
     people editing different windows would each see the other's save as "changing the plan"
     and get spurious structural conflicts once any concurrent activity was happening).
  2. **Openings and labels** - each level's `openings`/`labels` moved from an array to a map
     keyed by id, with each entry carrying its own `lastEditedAt`. On save, the Lambda merges
     key-by-key against what's stored: different keys touched by different people merge
     silently; the same key touched by both since this client's last load is a real conflict,
     collected into a `conflicts` array in the response (empty when everything merged clean).
- Deletes go through the same per-key comparison as edits, via a `{__deleted:true,
  lastEditedAt}` tombstone entry - so deleting an opening someone just edited is correctly
  flagged as a conflict too, not silently applied.
- A client resending an opening/label it never actually touched (happens every save, since the
  whole map goes out each time) is recognized as a no-op via a content-equality check, even if
  its baseline timestamp is stale - this is what keeps "different people, different openings"
  silent instead of spuriously conflicting.
- Backward compatible with plans saved under the old schema: they simply have no
  openingsByLevel/labelsByLevel/structuralLastEditedAt yet, which the merge code treats as
  empty/unset - no migration script, they migrate themselves on their next save.
- Verified the merge algorithm in isolation (6 scenarios: different-opening silent merge,
  same-opening conflict, stale-but-content-unchanged resend, delete-vs-concurrent-edit conflict,
  clean delete, brand-new key) - all behaved as intended.

### Round 9b-fix - closed a resurrection gap - BUILT, NEEDS REDEPLOY (supersedes the version above)
Found this while starting the frontend piece, before it shipped: the original merge deleted a
key outright once removed. But a client sends its whole openings/labels map on every save, so
if Person A loaded before Person B deleted a window, and A's next autosave landed before A's
5-minute poll caught up, A's unchanged resend of that window would look identical to "brand new
opening" to the merge logic - silently resurrecting something B deliberately deleted.

Fixed by keeping tombstones forever instead of removing the key - a `{__deleted:true,
lastEditedAt}` entry now stays in the map permanently once something's deleted, so the merge can
tell "never existed" apart from "existed, got deleted." A stale, unknowing resend of a now-deleted
item is treated as a conflict (surfaces to the person that it was deleted) rather than a silent
undo. Tombstones are a few bytes each - not a real cost concern at this scale.
Re-verified with 4 more scenarios (resurrection attempt correctly blocked, own-delete stale
resend is a harmless no-op, double-delete no-op, brand-new key still works) - all correct.

**ACTION NEEDED (you): copy the updated index.mjs into the Lambda console and hit Deploy again**
(this replaces what you just deployed - same table, same env vars, same routes, no other AWS
changes). Frontend work is paused until this redeploy is confirmed, since it needs to build
around the tombstone shape (skip __deleted entries when reading, understand `theirs: 'deleted'`
as a possible conflict shape) rather than the version without it.

**Do not push the frontend yet** - the frontend still sends the old whole-plan shape
(`data.levels[i].openings` as an array, single `expectedLastEditedAt`). The Lambda's PUT handler
now expects `openingsByLevel`/`labelsByLevel` as separate top-level fields and
`expectedStructuralEditedAt` instead. Once deployed, next step is adapting
stateForCloud()/loadCloudPlanIntoState()/syncToCloud() to the new shape (Round 9c, not started) -
holding off on the frontend push until that's built and tested, so beta doesn't briefly bounce
between mismatched schemas.

## Google OAuth (Sign-In With Google)
- Client ID: `729662824435-u239ge82b6kc7v4pqc0qrhloqqe0u6ga.apps.googleusercontent.com`
- Consent screen type: Internal (restricts to heartwoodrestore.com automatically)
- Authorized JS origins: [Amplify URL + custom domain if set]

## Region
- us-east-1 (all resources)

## DynamoDB
- Table name: Plans
- Partition key: planId (String)
- Capacity mode: On-demand

## S3
- Bucket name: heartwood-floorplan-photos
- Region: us-east-1
- Public access: fully blocked (private; access only via presigned URLs)
- CORS allowed origins: https://plan.heartwoodrestore.com, https://beta.heartwoodrestore.com

## IAM
- Lambda execution role: heartwood-floorplan-lambda-role
- Inline policy: heartwood-floorplan-data-access (DynamoDB CRUD+Scan on Plans table, S3 Put/Get on heartwood-floorplan-photos/*)

## Lambda
- Function name: heartwood-floorplan-api (deployed, single router-style function)
- Runtime: Node.js 22.x, handler index.mjs
- Execution role: heartwood-floorplan-lambda-role
- Env vars: TABLE_NAME=Plans, BUCKET_NAME=heartwood-floorplan-photos
- Timeout: 15s

## API Gateway
- API ID: rc57geq62e
- Invoke URL: https://rc57geq62e.execute-api.us-east-1.amazonaws.com
- Authorizer: JWT "google-auth", issuer https://accounts.google.com, audience = Google OAuth Client ID — attached to all 5 routes
- Routes: GET /plans, GET /plans/{planId}, PUT /plans/{planId}, POST /plans/{planId}/archive, POST /photos/upload-url
- CORS: origins plan.heartwoodrestore.com + beta.heartwoodrestore.com; methods GET,PUT,POST,OPTIONS; headers content-type,authorization
- Stage: $default, auto-deploy enabled

## AWS backend setup: COMPLETE

## Frontend integration: DONE (this pass)
- Google Sign-In button + status in header (auto re-signs-in on repeat visits via One Tap)
- App still loads straight into the last plan on this device (unchanged convenience behavior);
  blank plan if none. planId is generated client-side the moment a plan exists.
- "HWR Plans" button (shareBar, not "My Plans" - shop-wide, not personal) opens a searchable/
  filterable list of all cloud plans, click to load, Archive button per row, "+ New Plan" entry.
- Every local save (existing 500ms-debounced autosave) now also pushes to the cloud if signed in.
  IndexedDB remains the local safety net regardless of cloud/sign-in state.
- Optimistic-lock conflict handling: stale-data dialog lets you keep your version or load theirs.
- Status field editable via Options menu ("Status: Draft…").
- Archive (not delete) from the HWR Plans list.

## Known gap - deliberately deferred
Photos do NOT sync to the cloud yet - they stay device-local (as in v1) to avoid blowing past
DynamoDB's 400KB item size limit with embedded base64 images. The S3 bucket + presigned-URL
Lambda endpoint (POST /photos/upload-url) are already built and ready; wiring the photo capture
flow to use them is the next fast-follow, not done in this pass.

## Not yet done
- No local-to-cloud migration for old plans (per your call - starting fresh is fine)
- No Make.com/Airtable auto-archive webhook (planned fast-follow, needs airtableRecordId field added)

## Round 1 real-device testing (beta branch) - fixes applied
- Bug: "+ New Plan" silently did nothing. Root cause: doStartNew() was declared inside wireUp()'s
  closure, unreachable from the top-level HWR Plans dialog code - threw a ReferenceError in the
  console, closed the dialog, and stopped. Fixed by exposing it as window.doStartNew (same pattern
  already used for updateWindowDefaultBadge elsewhere in the file). Also dropped the redundant
  "save first?" confirmation for this entry point (redundant since it's a deliberate action from
  a list screen) and added a flush of any pending save before switching plans.
- UX change: removed the status filter chips row per your feedback. Replaced with a single
  "Show archived plans" checkbox - default view is active-only, checking it reveals archived
  plans with a Restore button. Required also updating the Lambda's listPlans() to stop excluding
  archived items server-side (now returns everything with an `archived` flag; filtering moved
  client-side).
- ACTION NEEDED: redeploy the Lambda - copy the updated index.mjs into the AWS console and
  Deploy again. The frontend fix requires no AWS changes, just re-pushing index.html to beta.

## Round 2 - menu cleanup + naming (per your feedback)
- Removed the status badge from each HWR Plans row entirely. The Status field itself (Draft/
  Sold/In Progress/Complete, via Options menu) is still there - only the list badge is gone.
  Flagged in case you want status dropped altogether, not just hidden from the list.
- New plans now require a name up front (prompted immediately on "+ New Plan" / File > New,
  before the blank canvas appears) - no more silent "Untitled Floor Plan" duplicates piling up
  in the cloud.
- Header simplified to exactly three buttons: File, Options, User.
  - File now includes "HWR Plans…" as its first item, and "Export…" collapsed PDF/CSV into
    one item with a sub-menu instead of two separate buttons.
  - User button: shows "Sign In" when signed out (tapping calls Google's prompt() directly -
    no more full-width rendered button/email crowding the header), shows "User" when signed
    in (tapping opens a menu titled with your email, with Sign Out).
  - Any action needing sign-in (HWR Plans) now shows a real dialog with a "Sign In" button,
    not just a toast.
- No AWS changes needed for this round - frontend only. Re-push index.html to beta and retest.

## Round 3 - full cloud transition, share links, Undo styling
- Removed Save/Open (local .json export/import) entirely, and the "Save first?" confirmation
  on New - both were v1 holdovers from when there was only one local save slot. Every plan now
  has a permanent cloud record, so New goes straight to the name prompt.
- Added real Share Link: File > Share Link puts a URL like
  https://plan.heartwoodrestore.com/?plan=<planId> on the clipboard (or native share sheet on
  mobile). Opening that URL loads that exact plan directly (prompts sign-in first if needed,
  then auto-opens it). This was in the original Phase 2 plan and hadn't been built yet.
- Export (PDF/CSV) unchanged - those stay since they're one-way deliverables, not app state.
- Undo button moved to the right of Saved/Synced status, styled orange (var(--accent)) for
  visibility.
- No AWS changes needed - frontend only, re-push to beta and retest.

## Round 4 - periodic "newer version available" check - BUILT
- Checks every 5 minutes, plus immediately whenever the tab/browser regains focus (Page
  Visibility API) - catches the common "switched apps for a while" case, not just a blind timer.
- Reuses the existing GET /plans/{id} endpoint - no new AWS/Lambda infrastructure needed.
- If nothing is unsaved locally, quietly catches you up to the latest cloud version with a toast.
- If you DO have unsaved local edits, it never touches them - just a toast + cloudIndicator
  nudge. Actual conflict resolution still happens through the existing 409/stale-data dialog
  when you next save.
- Skips the check entirely while a drag/gesture is in progress (activeDragCleanup set), so it
  can't interrupt active editing.
- Deliberately NOT true collaborative merging - see prior note, that's out of scope by design.
- No AWS changes needed - frontend only, re-push to beta and retest.

## Round 5 - photo cloud sync - BUILT
- Photos now travel between devices. Approach: each opening gets a new `photoKeys` field
  (parallel to the existing `photos` base64-preview field) holding the S3 key per slot.
- Capture flow unchanged for the person taking the photo (instant local base64 preview, same as
  always) - a new uploadPhotoToCloud() call fires alongside it, gets a presigned PUT URL from
  the existing /photos/upload-url endpoint, uploads the blob to S3, then records the key.
- On save, the plan's `data` blob still excludes base64 photos (DynamoDB size limit), but now
  includes a flat list of every photo's S3 key (collectPhotoKeys()) at the top level - reusing
  fields the Lambda already supported and never needed backend changes for.
- On load, resolvePhotoUrls() matches each opening's photoKeys against the fresh presigned view
  URLs the server returns (photoUrls, from the existing getPlan endpoint) and fills in o.photos
  so the existing display code (thumbnails, photo viewer, inline panel) needs zero changes.
- Removing a photo also clears its photoKeys entry so it stops syncing.
- Known limitation: if a photo is captured while signed out, it stays local-only until the next
  capture while signed in - no automatic retry/backfill for that case. Acceptable for now.
- No AWS changes needed - frontend only, re-push to beta and retest. Best test: capture a photo
  on one device/browser, then open the same plan on a second device (or private window, second
  Google account if available) and confirm the photo shows up there too.

## Offline-conflict discussion - only one change queued
Discussed the real risk of two techs offline on the same plan for 30-40 min each, then both
syncing - current conflict dialog is all-or-nothing (no partial merge), so whoever syncs second
can lose real work no matter which option they pick. Decided NOT to build true merging (matches
existing "no real-time sync" scope) and NOT to change the conflict dialog wording for now.
Persistent unsynced-work banner - BUILT. Shows a bright red full-width banner (reusing the
existing --danger color) when local edits have gone 10+ minutes without a successful cloud sync,
regardless of why (offline, signed out, sync failing). Tapping it retries the sync immediately
(or prompts sign-in if that's the blocker). Checks every 30s, tracks how long the current streak
of unsynced edits has lasted, resets automatically the moment a sync succeeds. Doesn't attempt
to fix the underlying conflict risk (no auto-merge, per the earlier discussion) - just makes it
visible in real time so people notice and can coordinate before it turns into a real conflict.
No AWS changes needed - frontend only, re-push to beta and retest. To test: disconnect wifi (or
sign out) mid-edit, keep drawing for 10+ min (or temporarily lower UNSYNCED_WARNING_THRESHOLD_MS
in the code for a faster test), confirm the banner appears and tapping it retries once back online.

## BUG FOUND on first retest - FIXED
Banner never appeared. Root cause: it was checking hasUnsyncedChanges() against
`lastStableSnapshot`, which updates on every LOCAL autosave (every 500ms) regardless of whether
the cloud sync succeeds - so it always looked "caught up" within 500ms even fully offline, and
the 10-min clock never started. Fixed with a dedicated unsyncedSinceAt timestamp, set the moment
an edit happens (in scheduleSave) and cleared only when a cloud sync actually succeeds (in
syncToCloud's success branch) or when loading/starting a fresh plan (loadCloudPlanIntoState,
doStartNew - so a stale timestamp can't carry over from a previous plan). The unrelated
lastStableSnapshot-based check (used by checkForNewerVersion for a different purpose - is it
safe to silently overwrite in-memory state) was left alone and renamed hasUnflushedEdits() for
clarity, so it doesn't get confused with the cloud-sync signal again.
Re-push to beta and retest - same test as before (offline or signed out for 10+ min while editing).

## Round 5 photo sync - CONFIRMED WORKING on retest
Bug found + fixed along the way: S3 bucket CORS had a typo in AllowedOrigins
("hearwoodrestore.com" missing the "t") - corrected, photo upload now working end to end.

## Window dragging jankiness ("screen shifts a bit") - DIAGNOSED + FIXED
Root cause: not the drag math, not touch-target size. Tapping a not-yet-selected window/door
(or label, or interior line/rect) to start dragging it calls updateInlinePanel() synchronously,
which toggles a `focus-mode` class on <body>. CSS hides the header, tabs, and toolbar entirely in
focus mode and reveals the bottom inline detail panel - a large layout change, not a small one.
Since #canvasWrap is a flex child sized by whatever's left after the header/tabs/toolbar/panel,
that whole reflow happened at the exact instant a finger touched down to begin a drag - the
canvas visibly resized right as the gesture started, so the drawing appeared to jump under the
user's finger before any real movement occurred. That's the "screen shifts" Nathan described, and
also explains the "too tight of a target" feeling - post-shift, the finger is no longer precisely
over the element it grabbed.

Fix: deferred updateInlinePanel() until the drag gesture actually ends (moved the call into the
onUp() completion handlers of startOpeningMoveDrag, startLabelMoveDrag, and
startInteriorMoveDrag, including the locked-level variant), instead of firing it immediately in
onPointerDown. The selection highlight itself (refreshSvg()) still fires instantly on touch-down
so you get immediate feedback that you grabbed the right thing - only the layout-changing panel
reveal is delayed. A simple tap-without-drag still shows the panel almost instantly, since onUp
fires right away when there's no real movement. During an actual drag, the canvas geometry now
stays completely stable for the whole gesture - no more mid-drag layout jump.
No AWS changes needed - frontend only. Re-push index.html to beta and retest: drag a window/door
across a wall and confirm the header/toolbar don't flicker/hide until you lift your finger, and
that the drawing no longer visibly jumps at the start of the drag.

## Round 6 - deeper dive on window/door dragging - BUILT
Follow-up after the first drag fix (deferred focus-mode panel reveal). Nathan reported two more
things: the layout shift still happens (now on release instead of drag-start), and dragging an
*already-selected* opening along the wall felt broken. Investigated both and built fixes for all
three root causes found:

1. Silent-deselect bug (the real cause of "can't drag along the wall while focused"): once an
   opening is selected, a decorative accent-colored highlight wash gets drawn on top of it along
   the whole wall band. That shape had no data-role, so most taps on the body of an already-
   selected opening were landing on nothing recognized and silently deselecting it instead of
   grabbing it to move. Fixed by making that highlight `pointer-events:none` so taps fall through
   to the opening's own (already-existing) generous move-hit-target underneath.
2. Resize handles were sitting exactly at the opening's two ends, competing with the move target
   for any tap near an edge - especially bad on narrower windows. Per Nathan's call (moving should
   be frictionless, resizing is rare and can take a little precision), handles are now smaller
   (r:11->7) and pushed 8 units past the opening's actual ends instead of sitting on top of them.
3. Layout-shift-on-focus (Nathan's zoom idea): instead of leaving the whole plan visible through
   the header/toolbar-hide reflow, selecting an opening/label/interior item now zooms in a
   conservative 1.3x centered on that item (reusing the same viewState the pinch-zoom/+/- buttons
   already drive), so the transition reads as "zooming in on what you tapped" rather than an
   unrelated jump. Exiting focus mode restores the exact pan/zoom you had before. If you'd already
   manually zoomed in further than 1.3x, it keeps your zoom level and just re-centers.
4. Slow edge auto-pan: while dragging a window/door (move or resize handle) near the edge of the
   now-zoomed-in view, the view slowly pans to follow so the item never runs out of visible canvas
   mid-drag. Deliberately gentle per Nathan's ask - a following-nudge, not a fast auto-scroll.
No AWS changes needed - frontend only, re-push index.html to beta and retest: select a window,
confirm you can immediately drag it again without it deselecting or grabbing a handle by mistake;
confirm the zoom-in on select feels smooth rather than jumpy; confirm dragging a window far along
a long wall slowly pans the view to follow instead of losing it off-screen.

## Round 7 - EXPERIMENTAL layout reorder (2026-08-01) - TRIED, ROLLED BACK
Nathan's verdict: still a large shift during focus mode, not worth the UI chaos of moving
tabs/title/level below the canvas. Reverted index.html back to the pre-experiment version
(confirmed byte-identical to index-backup-before-layout-reorder-2026-08-01.html). Header/tabs
are back above the canvas in their original order (shareBar, banner, header, tabs, views), and
shareBar + the unsynced banner hide in focus mode again as before.
Round 6's fixes (highlight-strip deselect bug, relocated/shrunk resize handles, focus-mode zoom
anchor, slow edge auto-pan) are untouched by this rollback and remain in place - those were a
separate, independent set of changes from the layout reorder and are still considered good.
Backup file kept around in case it's ever useful for reference, otherwise no longer needed.

## Round 7 (original entry, superseded by rollback above) - BUILT, not yet tested
Nathan's idea to further reduce the focus-mode shift: keep File/Options/User fixed at the very
top, put the canvas directly beneath it, then toolbar (already there), then Plan/Schedule tabs,
then the title/level/status/undo/rename/add-level block at the bottom.

Backup of the previous layout: index-backup-before-layout-reorder-2026-08-01.html (full copy of
index.html as it stood right before this change - if the new order doesn't work out, copy that
back over index.html to revert instantly, no need to hand-unpick anything).

What changed:
- Turned out toolbar was already directly under the canvas inside the Plan view - so this was a
  reorder of 3 top-level blocks (views, tabs, header), not a restructuring. New body order:
  shareBar, unsyncedBanner, views (canvas+toolbar or schedule table, whichever tab is active),
  tabs, header.
- shareBar and the unsynced-work banner no longer hide in focus mode (they used to, along with
  header/tabs/toolbar). Now that shareBar is the fixed anchor the canvas sits under, hiding it
  would defeat the point; the banner staying visible matches its "hard to miss" purpose even
  during a longer focus session.
- Header's divider line moved from border-bottom to border-top (it's now the last block, not the
  first) and its bottom row (level select / Rename / + Level) picks up
  env(safe-area-inset-bottom) padding, since it's now the true bottom edge of the screen on
  notched/gesture-nav phones - matches the same pattern already used for the inline panel and
  shareBar's top padding. Tabs picked up a matching border-top so there's still a visible divider
  between the canvas/schedule content and the tabs now below it.
- Toolbar and tab visual styling deliberately left untouched, so this test isolates just the
  ordering effect - worth a follow-up round of visual polish once Nathan has a feel for whether
  the order itself works.
No AWS changes needed - frontend only. Re-push index.html to beta and test on an actual phone
(this one really needs real-device testing, not just desktop resize) - specifically: does
selecting a window feel calmer now, do tabs/title/level controls feel reachable/sensible near the
bottom, does anything feel cut off near the screen edges.

## Round 8 - Crew Mode - BUILT, not yet tested
The full design from earlier discussion, now implemented. A UI display mode, not a real
permission boundary - matches the "roles are UI display modes" principle agreed early on.

How it's chosen: an explicit `?view=crew` or `?view=full` on the URL always wins and updates a
per-device remembered preference (localStorage key `floorplan-view-pref`); with no param, the
remembered preference is used; with neither a param nor a stored preference, it defaults to full
view - so normal day-to-day app use (bookmarked, no share link) is unaffected. Share Link now
always generates a crew-view link (`?plan=<id>&view=crew`) - no separate "share as full view"
option built, since a fellow manager would just open the app directly rather than via a link.

What's trimmed in crew view:
- No drawing toolbar. Since there's then no way to switch out of the existing "Move" tool, crew
  can still drag/reposition/resize existing windows and doors and drag walls (that capability
  was already built in, nothing new here) - they just can't add brand-new structural elements
  without switching to full view. This gives the "fallback, make changes on the fly" ability you
  wanted without any new permission-checking code.
- Level bar keeps the level dropdown (multi-story navigation still works) but drops Rename and
  + Level - those are setup actions.
- Title becomes read-only (no more tap-to-rename) - the plan is a shared record, renaming isn't
  part of day-to-day work.
- File menu keeps only Export and Share Link - drops HWR Plans (browsing every job in the shop),
  New (starting other plans), and Address… (already visible as a badge on the canvas and on the
  Schedule tab without a separate edit action).
- Options menu keeps only Manage schedule columns (useful, non-destructive) and a new
  "Switch to Full View" item - drops Status, Renumber, Lock Level, Reset Level, Delete Level.
- Full view's Options menu gains a matching "Switch to Crew View" item at the end, for previewing
  what crew sees without needing an actual share link.
- Lands on the Plan tab by default - this needed no new code, since a fresh page load has always
  started on the Plan tab regardless of view mode (ui.tab isn't persisted).

No AWS changes needed - frontend only. Re-push index.html to beta and test: generate a Share
Link, open it in a private window/second device, confirm it opens trimmed and lands on Plan;
confirm "Switch to Full View" restores everything and is remembered on that device even after
closing and reopening the tab; confirm your own normal (non-shared-link) usage still opens full
view as always.

## Round 9c - Frontend: adapt to the keyed-map shape - BUILT, not yet tested
Internal app state is unchanged - lvl.openings/lvl.labels are still plain arrays everywhere
(drawing, dragging, schedule, PDF export, all of it). The conversion to/from the cloud's map
shape happens only at the sync boundary:

- Each opening/label now carries a `lastEditedAt` field (defaults to null when created or when
  restored from data that predates this). It's the per-key baseline from Decision 2 - not shown
  anywhere in the UI.
- `lastKnownKeysByLevel` - a new module-level snapshot of {levelId: {openings:{id:lastEditedAt},
  labels:{id:lastEditedAt}}} captured after every load and every successful save
  (captureKnownKeysFromState()). This is how a local deletion gets detected: an id present here
  but missing from the current array gets a {__deleted:true, lastEditedAt} tombstone synthesized
  when building the next save's payload.
- `stateForCloud()` renamed to `buildCloudPayload()` - now returns `{data, openingsByLevel,
  labelsByLevel}` instead of just a data blob. `data.levels[i]` no longer carries
  openings/labels arrays; those travel separately, keyed by level id then item id.
- `syncToCloud()` sends the new shape and `expectedStructuralEditedAt` (was
  `expectedLastEditedAt`) for the whole-plan structural gate.
- New `applySyncResult(item, conflicts)` reconciles the PUT response back onto local state -
  critically, flows the server-assigned lastEditedAt back onto each opening/label (skipping this
  would make every next save think its baseline is stale and falsely conflict with itself),
  removes anything the server confirms is gone, and re-captures the known-keys baseline.
- **Interim conflict handling** (placeholder for Round 9d's real dialog): when the response
  includes conflicts, this applies whichever version the server kept and toasts that it
  happened ("their version was kept, redo yours if still needed") - not the scoped one-at-a-time
  "keep mine / keep theirs" dialog from Decision 3 yet, but safe: nothing loops forever trying to
  resave a rejected change, and nothing is silently lost without the user finding out.
- `handleSaveConflict()` (the 409 dialog) now only fires for STRUCTURAL conflicts - renamed
  fields to match (structuralLastEditedAt), same "keep mine / discard mine" choice as before.
- `loadCloudPlanIntoState()` gained `applyServerOpeningsAndLabels()`, which reconstructs the
  array shape from the cloud's map shape (skipping deletion tombstones) - with a fallback that
  does nothing if a plan predates this change entirely, in which case whatever applyLoadedState
  already restored from the old array shape is used as-is (migrates itself on next save, same
  backward-compat approach as the Lambda side).
- Verified the full round trip in isolation (first save, clean resend with zero conflicts,
  edit+save with correct lastEditedAt reconciliation, delete+tombstone, and a stale second
  client's resurrection attempt correctly blocked as a conflict) - all 5 scenarios came out right.

**Ready to push to beta and test** - this is the checkpoint: confirm normal single-device
save/load still works with zero visible difference, and that two people editing different
openings on the same plan merge silently with no interruption. True same-opening conflicts will
work (server-side) but the UI response is still the interim toast-and-overwrite, not final -
that's Round 9d next, not built yet.

## "Window / Door Schedule" renamed to "Window Schedule" - BUILT
Renamed everywhere it appeared: the tab label and both PDF export page headers. No AWS changes -
frontend only, re-push to beta and retest along with the banner fix.

## Banner dismiss lag - FIXED
Banner correctly appears now. Small polish issue Nathan noticed: after "Synced" shows, the
banner took a few seconds to disappear instead of clearing immediately. Root cause: it only
re-checked its own visibility on the 30-second setInterval tick (startUnsyncedBannerTimer), not
the moment syncToCloud() actually succeeded. Fixed by calling updateUnsyncedBanner() directly
right after `unsyncedSinceAt = null;` in three places: syncToCloud's success branch,
loadCloudPlanIntoState, and doStartNew - so it reacts immediately instead of waiting for the next
timer tick. No AWS changes needed - frontend only, re-push to beta and retest.

## Unsynced-banner threshold - discussed, staying at 10 minutes
Confirmed the threshold is a pure local check (no AWS cost at any value) - the real tradeoff is
usability, not cost. Shorter risks false alarms from routine job-site connectivity blips, which
would train people to ignore it right when it matters most. Decision: leave at 10 minutes,
revisit later based on real field data once crew is actually using it, rather than guess further
from a desk. No code change made.

## Round 9d - two-phone testing found real bugs in the structural gate - FIXED, deploy needed
Nathan tested Round 9c on two real phones (same account, opposite walls, adding then editing
openings). Two real bugs found, both in `index.mjs`, not the merge logic itself:

1. **Frontend leak**: `buildCloudPayload()` cloned the entire `state` object into `data`,
   including `state.lastEditedAt` and `state.structuralLastEditedAt` themselves - both change on
   every save from either device, so they were embedded in the exact blob the Lambda diffs to
   decide "did anything structural change." Any save from one device made the *other* device's
   next save look structurally different, regardless of what was actually touched. Fixed: both
   fields are now deleted from the clone before it's sent - they're tracked and restored
   separately at the top level already (confirmed against `loadCloudPlanIntoState`, which sets
   them from `rec.lastEditedAt`/`rec.structuralLastEditedAt` directly, never from `data`).

2. **Lambda architecture bug** (the bigger one): the structural gate did `return json(409, ...)`
   *before* the per-opening/per-label merge ever ran. So any structural baseline mismatch -
   real or false-positive - blocked opening/label merges too, contradicting the "two independent
   mechanisms" design. This is what made "Keep mine"/"Discard mine" look like it was arbitrarily
   picking winners: opening edits were queuing up unsynced behind unrelated wall disagreements,
   and a forced resync just happened to push through whatever full snapshot was on that phone at
   that moment. Also, the gate fired on baseline mismatch alone, without checking whether the
   structural content actually differed.

   Fixed by refactoring `savePlan` into a pure `computeSave(planId, prev, body, userEmail, now)`
   (no I/O, unit-testable) plus a thin I/O wrapper. The per-opening/label merge now always runs,
   independent of the structural check. A structural conflict now requires BOTH a stale baseline
   AND actually-different content. When a genuine structural conflict exists, only the structural
   half of the write is withheld (`data` and `structuralLastEditedAt` keep the previously stored
   values) - the opening/label merge is written either way, so nobody's window/door edits wait on
   an unrelated wall disagreement. The 409 response's `current` field now includes the
   already-merged openings/labels, so even "Discard mine" doesn't throw away opening edits that
   had nothing to do with the structural conflict.

   `computeSave`/`mergeKeyedMap`/`openingContentEqual` are now exported (harmless for the Lambda
   runtime - `handler` is still the only entry point AWS calls) so they can be tested directly.

**Verified with a scripted test harness** (`test-conflict-scenarios.mjs`, left in the outputs
folder - run with `node test-conflict-scenarios.mjs` after `npm install` for the AWS SDK
packages, or stub them out for a network-free run). 20 assertions across 6 scenarios, all passing:
different-openings silent merge, the exact bug Nathan hit (opening edit + concurrent real wall
edit - opening edit no longer lost, structural conflict still correctly flagged), stale-baseline-
but-identical-content (no false conflict), true same-opening conflict (still correctly flagged,
first-applied value kept), old-schema backward compatibility, and deletion-tombstone
resurrection-blocking. `node --check` clean on both files.

**Scope decision (Nathan's call, agreed)**: don't build the full one-at-a-time same-opening
conflict dialog (Round 9d's original scope per the design doc) right now. Existence races
(add/delete during someone else's edit) are already safe via tombstones - worst case is
last-write-wins, never corruption or resurrection. True same-field-same-opening collisions will
be rare in real crew use; the existing interim behavior (silently keep whichever saved first,
toast the loser) is an acceptable stopgap for that rare case. Effort instead went into fixing the
above two bugs, since making the common case - different crew members, different openings, no
interruption - actually reliable matters more than polishing the rare-conflict UX right now.

**Needs deploy + retest**: `index.mjs` needs to be redeployed (frontend `index.html` fix from
earlier this session too, if not already pushed). Retest recipe: same two-phone setup, edit
different openings' details (not add/delete) on opposite sides - should now merge with no dialog
at all. Then, to specifically confirm the fix, have one phone make a real wall edit while the
other edits an unrelated opening at the same time - the wall-editing phone's disagreement should
show the structural dialog as before, but the *other* phone's opening edit should go through
cleanly without ever seeing it.

## Round 9e - two more bugs from live retest (deployed Round 9d Lambda) - FIXED, deploy needed
Nathan retested on two phones with the fixed Lambda actually confirmed deployed (pasted the exact
file back to verify). Both phones got the structural conflict dialog immediately on sign-in
before either made an edit, and a genuine field edit (a note changed from "Hi" to "Hi there")
silently reverted back to "Hi" on both phones about 5 minutes later, after the periodic refresh.

1. **JSON.stringify is key-order-sensitive, and key order isn't stable here.** `structuralChanged`
   compared `body.data` against `prev.data` with `JSON.stringify(...) !== JSON.stringify(...)`.
   But `loadCloudPlanIntoState()` ends by calling `doSave()`, which immediately re-syncs to the
   cloud right after every load - and the client rebuilds that data via migration
   functions/Object.assign on the way in, which does not preserve the original key order. So a
   plan that hasn't structurally changed at all could still stringify differently than what's
   stored, purely from key order - producing a fresh structural conflict on essentially every
   load. This is almost certainly why the dialog appeared immediately, on both phones, before
   either had done anything. Fixed with a proper order-independent `deepEqual()`, used for both
   `structuralChanged` and `openingContentEqual`'s per-field comparison (same risk applies to any
   opening field that's itself an object, e.g. `cells`).

2. **The periodic "newer version" check was using the wrong signal to decide if it's safe to
   silently overwrite local state.** `hasUnflushedEdits()` compared against `lastStableSnapshot`,
   which settles to match current state within 500ms of any edit - regardless of whether the
   cloud sync that follows actually succeeds. So the moment Phone 2's note edit hit a conflict and
   got abandoned (dismissed via the dialog's Cancel/backdrop), the app had already privately
   marked it "safe," even while the cloud indicator correctly showed "Conflict." Five minutes
   later, `checkForNewerVersion()` saw nothing at risk and silently pulled the server's copy over
   it. Fixed: `hasUnflushedEdits()` now checks `unsyncedSinceAt !== null` instead - the same signal
   that already correctly drives the 10-minute unsynced-work banner, cleared only on a genuine 200
   from `syncToCloud()`. A stuck/abandoned edit is now protected from being silently discarded and
   will surface via the existing banner instead.

Didn't make the structural conflict dialog non-dismissable - with fix #2 in place, dismissing it
is no longer lossy (the edit stays protected locally until it's actually resolved), so forcing an
explicit choice isn't necessary right now.

**Verified**: extended `test-conflict-scenarios.mjs` to 29 assertions (added a same-content-
different-key-order scenario proving no false structural conflict, plus direct `deepEqual` sanity
checks) - all passing. Both files syntax-clean.

**Needs deploy + retest**: `index.mjs` redeploy + `index.html` push. Retest recipe: open the plan
fresh on both phones - should NOT see the dialog immediately anymore. Then repeat the
different-opening-edit test from Round 9d's retest recipe.

## Round 9f - real data-loss bug: concurrent saves could silently erase each other - FIXED
Nathan's retest of Round 9e (dialog-on-load bug fixed, confirmed) surfaced a genuinely different,
more serious bug: phone 1 added a door, phone 2 edited a note on a different window at roughly
the same time - no dialog either time - but the door was never merely *not shown* on phone 2, it
was actually gone from the database. Refreshing, waiting 5+ minutes, switching tabs - nothing
brought it back. Only opening a different plan and returning (a full reload) recovered it, which
was the tell that this wasn't a display/refresh problem.

**Root cause**: `savePlan` was a plain read-merge-write - GetCommand the current record, compute
the merged item in memory, PutCommand the whole thing back. That's a classic lost-update race.
The per-opening merge only knows about whatever THIS request's own GetCommand happened to see. If
phone 1's read landed before phone 2's write completed, phone 1's own PutCommand still carries
forward its own now-stale copy of the *entire* openingsByLevel map - including phone 2's window,
which phone 1 never touched and still has the old content for. From phone 1's point of view
nothing about that key changed, so it gets carried forward unchanged into the write - silently
overwriting whatever phone 2 had just added, including keys phone 1 never even looked at (the
door). Nothing in the merge logic itself was wrong - the problem was two independent writers each
trusting their own snapshot as ground truth for a full-map replacement.

**Fix**: optimistic concurrency on the write itself, not just on the structural field.
`savePlan` now retries against a fresh read whenever the write's `ConditionExpression` (item's
`lastEditedAt` must still match what this request read) fails - i.e. whenever another save landed
in between. Refactored into a `savePlanWithStore(store, ...)` that takes a storage abstraction
(get + conditional put), so the retry loop itself is unit-testable against an in-memory fake
without a real DynamoDB table. No new IAM permissions needed - conditional writes use the same
PutItem action as before.

**Verified**: added a scripted race to `test-conflict-scenarios.mjs` that reproduces this exact
scenario deterministically (phone 2's full save runs to completion in the middle of phone 1's
read-then-write window) - confirms both the door and the note survive. 32 assertions total, all
passing. Both files syntax-clean.

**Needs deploy + retest**: `index.mjs` only this round (no frontend change). Retest recipe: same
as before - two phones, each edits a different opening at close to the same time, no dialog
expected either way, and now BOTH edits should show up on BOTH phones without needing a full plan
reload to recover.

## Round 10a - Opening Status Indicators & Issue Tracking: Batch A (Lambda) - BUILT, deploy needed
First of three batches for the feature described in phase2b-conflict-resolution-notes.md's
"Opening Status Indicators & Issue Tracking" section. This batch is backend-only: gives the
per-opening merge logic the special-case behavior `issues` needs before any frontend UI is built
on top of it.

`issues` (per-opening issue log entries: {time, user, message}) needs union-by-content semantics,
not last-write-wins - two crew members each adding a note to the same window in the same sync
window must both survive, not just whichever save lands last. `mergeKeyedMap` now carves this out:
`issues` always unions (deduped by time+user+message) regardless of what else happens with that
key, while every other field on the opening keeps the exact same last-write-wins/conflict behavior
as before - including when there's a genuine conflict on some other field at the same time, the
issues union still goes through untouched by that conflict. `openingContentEqual` gained an
optional extraExclude param so `issues` can be left out of the "is this key actually changing"
check without touching its existing behavior for every other caller. Labels don't have `issues` -
fully backward compatible no-op for them.

**Verified**: two new scenarios in `test-conflict-scenarios.mjs` (two devices each adding a
different note to the same opening - merges silently, no conflict; issues unioning even alongside
a genuine conflict on a different field of the same opening) plus direct `unionIssues` checks -
43 assertions total, all passing. Syntax-clean.

**Needs deploy**: `index.mjs` only, no frontend changes yet - nothing in the app actually writes
to `o.issues` until Batch C builds the UI for it, so this is safe to deploy standalone with zero
visible change in the meantime.

**Also queued for Batch B** (per Nathan): in crew view, "Window Number", "Completed", and "In
Progress" columns should always show and not be optional to hide - `state.columnVisibility` is a
shared/synced setting, so a manager hiding one of these for full view would otherwise also hide it
from crew. Plan: `visiblePresetColumns()` forces these three visible whenever
`ui.viewMode==='crew'` without mutating `state.columnVisibility` itself (stays fully optional in
full view, as today), and `showColumnsDialog()` shows them as locked/non-interactive rows in crew
view instead of removing them from the list.

## Round 10b - Opening Status Indicators: Batch B (frontend visuals + crew locked columns) - BUILT
Frontend-only, no Lambda change. `index.mjs` from Batch A already deployed, no further backend
work needed for this piece.

- **Completed checkmark**: no schema change needed - `completed` was already a checkbox column.
  Added a small corner badge (green circle + checkmark) on the opening's number badge on the plan
  canvas when checked, matching the existing `.tbtnBadge` small-pill visual language.
- **In Progress**: converted from a checkbox to a dropdown (`kind:'checkbox'` -> `kind:'dropdown'`
  with a blank/25%/50%/75%/90% option list) - this reused the schedule's existing generic dropdown
  rendering everywhere (schedule table, inline panel, PDF/CSV/print export all already branch on
  `col.kind` generically, nothing hardcoded the old checkbox assumption). Renders as a small
  number overlay in the same corner slot the checkmark uses - the two are mutually exclusive by
  design, so they never compete for space. Migration: any opening with an old boolean `inProgress`
  value gets it cleared to blank on load (a boolean can't map to a specific percent - the crew
  re-picks a real value next time they touch that opening).
- **Mutual exclusivity**: checking Completed clears the In Progress percent (both the schedule
  table's and inline panel's checkbox handlers now special-case `col.key==='completed'`). Not
  built the other direction (picking a percent doesn't uncheck Completed) - wasn't asked for and
  the checkmark-clears-percent direction is the one that actually matters for avoiding a
  contradictory-looking icon.
- **Crew-mode locked columns** (added to this batch per Nathan): "Window Number", "Completed", and
  "In Progress" now always show in crew view regardless of the shared `columnVisibility` setting -
  `visiblePresetColumns()` forces them visible when `ui.viewMode==='crew'` without mutating the
  underlying preference, so full view stays exactly as optional as before. `showColumnsDialog()`
  shows these three as checked-and-disabled rows (with a small "(always shown in crew view)" note)
  instead of removing them from the list entirely, so it's clear why they can't be unchecked.

Syntax-checked clean. No new scripted tests this round (pure UI rendering + a data-shape
migration, not merge logic) - worth a real-device pass: check a window's checkmark/percent badge
appears on the plan canvas, toggling Completed clears an existing In Progress %, and that crew
view can't hide Window Number/Completed/In Progress from Options -> Manage schedule columns.

Next: Batch C, the issue tracker UI itself (glow + comment badge, add-entry log, Resolved with
confirm).

## Round 10c - rapid-add windows disappearing - FIXED (Nathan's diagnosis was right)
Nathan noticed: adding windows quickly around a wall, some added after the first would vanish
shortly after. Correctly guessed it was overlapping syncs and that a sync queue was the fix.

**Confirmed mechanism**: `syncToCloud()` had no protection against firing a second request while
the first was still in flight - each debounced save just fired its own PUT. `applySyncResult()`
rebuilds a level's `openings` array from whatever THAT response's `openingsByLevel` says exists,
filtering out anything not present. If window 2 gets added and synced while window 1's request is
still in flight, and window 1's response happens to arrive AFTER window 2's (entirely possible -
plain network jitter, or window 1's request needing a retry per the Round 9f lost-update fix),
that late-arriving response reflects server state from before window 2 was even sent - and
applying it wipes window 2 back out of local state, even though the server itself never lost it.

**Fix**: `syncToCloud()` now serializes itself - only one PUT in flight at a time. A save
triggered while one is already in flight just sets a flag instead of firing an overlapping
request; the in-flight request's completion (success, conflict, or failure) triggers exactly one
more sync afterward, picking up everything added since. Every request now always carries the full
current state, and responses are always processed in the order they were sent, so this class of
bug can't happen for any field, not just openings - same fix, no separate cases needed.

Frontend-only, no Lambda change. Syntax-checked clean. No new scripted test (this is client-side
request sequencing, not server merge logic) - worth confirming on a real device: add several
windows rapidly around a wall and make sure none of them flicker away.

## Round 10d - rapid-add windows still flickering after Round 10c - FIXED (the real gap)
Round 10c's serialization (never two requests in flight at once) helped but didn't fully fix it -
Nathan still saw it happen at a steady fast pace ("about the beat of Another One Bites the Dust",
~110-120bpm, so roughly one add every ~500-550ms - right at the edge of the 500ms debounce).

**The actual remaining gap**: serializing requests prevents *overlapping* ones, but a single
request can still take a few hundred ms to round-trip. If a window gets added locally WHILE an
earlier request is already in flight (easily possible even one-at-a-time, just from normal
network latency), that in-flight request's payload was built before the new window existed, so it
has no idea about it. `applySyncResult()` was rebuilding each level's array from "whatever this
response says exists" and discarding anything else - so when that in-flight response arrived and
got applied, the window added in the meantime looked "not present" and got filtered straight back
out of local state, even though it was never sent to the server yet, let alone rejected by it.
Round 10c's fix only closed the door on overlapping requests; it didn't change what happens when a
single request's response arrives after newer local edits have already happened, which is the
much more common case at any real editing speed.

**Fix**: `applySyncResult()` now takes the exact `openingsByLevel`/`labelsByLevel` that request's
own payload contained (captured at request-build time). An opening only gets removed from local
state if it was part of THAT SPECIFIC request's payload and the response confirms it's gone
(deleted or missing). Anything not part of that request's payload - i.e. added locally after the
request was already built - is left alone regardless of what the response says, since the
response simply never had an opinion on it. The next sync (already correctly queued/serialized
per Round 10c) picks it up normally.

Frontend-only, no Lambda change, single call site updated (`syncToCloud()`'s success handler).
Syntax-checked clean. Worth retesting at the same pace as before - should hold up now regardless
of add speed, since this closes the actual gap rather than just reducing how often it's hit.

## Round 10e - Opening Status Indicators: Batch C (issue tracker UI) - BUILT
Frontend-only, no Lambda change (Batch A's union-merge from earlier already handles the sync side).

- New `o.issues` array (`{time, user, message}` entries), defaulted to `[]` on creation
  (`addOpening`) and on migration (`migrateOpeningToV3`, for plans saved before this). French Pair
  conversions start both new openings with an empty log rather than inheriting the original's -
  ambiguous which of the two should get it, and they're new ids anyway.
- **Canvas indicator**: red/pink glow behind the opening's number badge, plus a small red corner
  badge (three white dots, chat-bubble style) in the opposite corner from the completed/in-progress
  badge, so both can show at once - independent of completion status, per the design doc. The
  corner badge is its own tap target (`data-role="issuesBadge"`, checked ahead of the opening
  itself in `onPointerDown` since it's nested inside the opening's badge group) - tapping it
  selects the opening and opens the issue log directly, rather than just selecting.
- **Issue log dialog** (`showIssueLogDialog`): entries listed newest-first with who/when, a text
  input + Add button, and (only shown when there's at least one issue) a "Resolved (clear all)"
  button gated behind a danger-styled confirm dialog matching the existing reset-level/delete-level
  pattern. Closing re-syncs the inline panel so its issue count stays accurate.
- **Inline panel entry point**: a new "Issues" row (added since the canvas badge can be a small
  target to hit precisely, especially zoomed out) - shows the open count and opens the same dialog,
  styled as a plain button when empty ("No issues – add one") or a danger-styled button with the
  count when not.

Syntax-checked clean. No new scripted tests (pure UI, the merge semantics were already verified in
Batch A). Worth a real-device pass: add an issue from the canvas badge and from the inline panel,
confirm the glow/badge appears and disappears correctly, and that Resolved actually requires the
confirm step.

This closes out the "Opening Status Indicators & Issue Tracking" feature from
phase2b-conflict-resolution-notes.md (Batches A/B/C all built and, per Nathan, working).

## Round 10f - status/issue indicators enlarged and moved to the opposite side - BUILT
Nathan confirmed the small corner badges from Batches B/C were too small to read on a phone.

Moved completed/in-progress/issues off the number badge entirely and onto a second, near-full-
size badge (radius 14, vs. the number badge's 15) on the OPPOSITE side of the opening. Reused
`badgeNormal` (already correctly flipped for doors to sit clear of the swing arc) negated, so the
new badge automatically lands on whichever side the number badge *isn't* on - for windows and
non-operating doors that's the side opposite wherever the wall's outward normal put the number
badge; for operable doors specifically, it means this indicator can end up sitting over the swing
arc. Confirmed with Nathan this is an acceptable tradeoff - legibility of the indicator matters
more than keeping the arc visually clear.

Visual logic on the new badge:
- Completed -> green circle, white checkmark (same checkmark path helper, now parameterized with
  a size argument so it can scale up).
- In Progress (no Completed) -> orange circle, white percent number.
- Issues with neither of the above -> red circle, three white dots as the main glyph (was
  previously the only issues glyph, now just bigger).
- Issues alongside Completed or In Progress -> the primary color/glyph stays, plus a small red
  corner accent (dot-badge) on the big circle so the issue is still visible without competing with
  the primary status.
- Red glow behind the whole badge whenever there are open issues, regardless of what else is set.
- Only tappable (opens the issue log) when there are actually issues to show - same behavior as
  before, just relocated. No badge renders at all when none of the three apply.

French Pair openings needed no special handling - each leaf is already rendered as its own
independent opening with its own wall position/badgeNormal, so each gets its own correctly-placed
status badge automatically.

Frontend-only, syntax-checked clean. Worth a real-device look, especially: readability at a
normal zoom level, and a door where the indicator now covers the swing arc - confirm that's
acceptable in practice, not just in theory.

## Session paused here - queue for next time
- Retest this round's three fixes together on beta: banner dismiss lag, "Window Schedule"
  rename, and window/door/label/interior-item drag jankiness (see sections above for each).
- Retest rounds 3 and 4 on beta (Save/Open removal, Share Link, Undo styling, periodic refresh
  check) - not yet confirmed on real devices. Share Link specifically confirmed to exist and
  needs real-device verification (?plan=<id> URL, opens that plan directly, prompts sign-in
  if needed).
- User button now shows a small profile icon (SVG person glyph) instead of the word "User" when
  signed in - "Sign In" text unchanged when signed out. Small tweak, built, needs retest too.
- BUG FOUND + FIXED: signed-out "Sign In" button stopped working after the One Tap prompt was
  dismissed/missed once - Google suppresses auto-showing One Tap after a dismiss, and our custom
  button was just calling that same suppressed prompt(). Fixed by rendering Google's own small
  icon-only button (type:'icon') into the signed-out slot instead - a genuine user click on
  Google's real widget isn't subject to that suppression, unlike our own prompt()-calling button
  was. requireSignInDialog updated too (no longer tries to trigger sign-in itself, just points
  at the real button). No AWS changes - frontend only, re-push to beta and retest, specifically:
  dismiss/miss the prompt once on purpose, then confirm the icon button still works afterward.

## Crew Mode - open design question, NOT built yet
Nathan's goals (his words): crew needs a fallback/ability to make on-the-fly changes, but should
naturally gravitate to a simplified "crew mode" by default for their real workflow (mark
completed, take notes/photos) - with an always-available menu escape hatch to full management
mode. Accepted tradeoff: once in management mode, crew has full destructive access, same as the
existing "roles are UI display modes, not permission boundaries" principle from early on.

Claude's proposed approach (discussed, not yet built or agreed):
- The ?view=crew URL param (planned since the original Phase 2 schema design, never built) is
  the entry point - Share Link gets a toggle to generate either a normal or a crew-view link.
  Crew mode becomes the *default* naturally because it's literally what their link opens into,
  no choice required from them.
- Crew mode lands on the Schedule tab (not Plan/drawing), hides the drawing toolbar and reduces
  header clutter (Options, Address edit) - optimized for checking off completion/flags/notes/
  photos per opening, which is their actual day-to-day task.
- One always-visible menu item to switch to full Management Mode - no confirmation, no gate,
  immediate full toolset (matches the no-permission-boundary principle).
- Open question for Nathan to weigh in on: should a device "remember" the last mode someone
  manually switched to (via localStorage), or always reset to crew mode when reopening a crew
  link fresh? Leaning toward remembering per-device so a crew member who regularly needs full
  access isn't fighting the UI every time - but this is Nathan's call, not decided yet.
- Backlog, roughly in priority order: Crew Mode (BUILT in Round 8 above, needs real-device
  testing), per-opening color tag/flag/comment, Make.com/Airtable auto-archive webhook (needs
  airtableRecordId field + a machine-auth endpoint), then promote beta to production once
  everything's solid.
