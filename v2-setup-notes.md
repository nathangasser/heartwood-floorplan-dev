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

## Round 10g - deleted windows silently coming back on reload - FIXED + delete now confirms
Nathan deleted several windows via the schedule's ✕ button, plan looked clean, but reloading from
the cloud brought a cluster of them back (some with duplicate-looking numbers, since new windows
had reused numbers the "deleted" server-side entries still occupied).

**Root cause**: `applySyncResult()` was calling `captureKnownKeysFromState()` - a full blind
re-snapshot of whatever's currently in local state - after every successful sync response. If a
deletion happened locally *after* an earlier, unrelated request was already sent but *before*
that request's response came back, the blind re-snapshot would "forget" the deleted opening had
ever existed before its own tombstone was ever actually sent to the server. The next sync's
tombstone diff then had nothing to compare against, so it never mentioned that opening at all -
the delete silently never reached the database, even though it looked gone locally. This is the
same underlying class of bug as Round 10d (a response arriving after newer local activity), just
hitting the deletion-tracking bookkeeping instead of the openings array itself - realistic to hit
exactly when deleting several windows in a row, since each deletion's own sync can easily still be
in flight when the next one starts.

**Fix**: replaced that blind re-capture with `reconcileKnownKeysAfterSync()`, which only updates
bookkeeping for keys that were part of THAT SPECIFIC request's own payload (using the same
sent*ByLevel captured at request-build time from Round 10d). Anything not part of that request -
an addition or deletion made after it was already sent - is left completely untouched, so a later
sync can still correctly detect and report it. `captureKnownKeysFromState()`'s other call sites
(full plan load, new plan, initial local-storage restore) are unaffected and correctly still do a
full re-snapshot, since those really are establishing a fresh baseline from scratch.

**Note for Nathan**: this only prevents it going forward - "Test 10 Floor Plan" already has the
zombie windows baked into its stored record from before this fix. Deleting them again with the
fix deployed should make it stick this time; no separate cleanup script needed.

**Also added**: the schedule's ✕ delete button now goes through a confirm dialog first (same
danger-styled pattern as level reset/delete), instead of deleting immediately on tap with zero
feedback.

Frontend-only, no Lambda change. Syntax-checked clean. Worth retesting the exact scenario: delete
several windows in a row, wait for Synced, then do a full reload (not just switching tabs) and
confirm they're actually gone.

## Round 10h - remove opening delete buttons from crew view - BUILT
Hidden entirely (not just confirmed) in crew view, same principle as the rest of crew mode's
trimming - deleting an opening is a shop-management action, not part of the day-to-day workflow,
and crew's escape hatch is Switch to Full View if they genuinely need to:
- Schedule row's ✕ delete button.
- Inline panel's ("focus view") "Delete this item" button.

Also noticed while in there: the inline panel's delete button had no confirm dialog at all (unlike
the schedule's, fixed last round) - added the same danger-styled confirm + toast to it for full
view, so both delete entry points now behave consistently.

Frontend-only, syntax-checked clean. Worth confirming on a real device: crew view shows neither
delete button anywhere for openings, full view still has both (now both behind a confirm).

## Round 10i - landing chooser instead of silent blank plan - BUILT (frontend only)
Root cause of abandoned/untitled plan clutter: on any fresh device/browser storage (first-ever
load, cleared site data, private browsing, Safari purging IndexedDB after inactivity - all things
that happen a lot during two-phone testing), init() had no local plan to restore, so it just fell
through to the default blank state (title "Untitled Floor Plan") and immediately assigned it a
real planId. From that point it's a live plan - the next autosave silently creates a permanent
DynamoDB record, with no prompt or name required.

Note: the explicit File > New... flow already required a name (promptNewPlanName/
startNewPlanFlow, built earlier) - that part didn't need building. The gap was specifically the
silent landing path.

Fix:
- loadState(cb) now reports whether it actually found and restored a saved plan (cb(foundLocal)),
  instead of always calling cb() with no signal either way.
- checkUrlForSharedPlan() sets a new sharedPlanRequested flag whenever the URL had ?plan= at all
  (whether it resolved immediately or is pending sign-in) - keeps the new landing chooser from
  double-firing on top of an intentional shared-link open.
- New showLandingChooser(): if signed in, opens the existing HWR Plans dialog (already has search
  + "+ New Plan", which already requires a name); if not signed in yet, sets pendingLandingChooser
  and waits - handleGoogleCredential retries it after sign-in completes, same pattern already used
  for a pending shared-link plan.
- init()'s loadState callback calls showLandingChooser() when !foundLocal && !sharedPlanRequested.

Low-risk by design: the blank state still renders underneath (nothing about init() otherwise
changed), and nothing autosaves to the cloud until the user actually edits something - so even if
someone dismisses/closes the picker without choosing, they're no worse off than before (lands on
a blank draft, same as today), they just get the choice surfaced instead of it happening silently.

Frontend-only, syntax-checked clean. Worth confirming on a real device: clear site data (or use
a private window) and reload - should land on the HWR Plans picker instead of a blank plan;
signing in from a cold start should also trigger the picker right after sign-in completes; a
shared link (?plan=...) should still go straight to that plan with no picker in the way.

## Round 10j - dragging any wall shoved unrelated openings toward a corner - FIXED
Nathan caught this with real before/after screenshots: nudged one wall near W14, and D2 (a French
door pair on a completely different, untouched wall several edges away) visibly slid from ~58%
along its wall to ~90% - toward the corner. "Plenty of other windows" moved too.

Root cause: clampOpeningsToWalls() runs on every pointermove tick of a wall drag, and reprocesses
EVERY wall in the level that has openings, not just the two adjacent to whatever's being dragged.
For any wall outside the drag's frozen gap-baseline, it recomputed that wall's opening positions
from scratch via rackOpeningsOnWall() - but that function's gap model only tracks the space BEFORE
each opening, never the trailing gap after the last one on a wall. Re-deriving "before" gaps from
current offsets and reapplying them silently swallowed any real trailing gap into those before-
gaps, yanking the whole opening group toward the far end of the wall - on every wall in the plan,
on every single drag tick, regardless of which wall was actually being resized. Hand-checking the
math against Nathan's own numbers (58% -> ~90%) landed right in line with what the formula
predicts for a wall with a sizeable trailing gap.

Fix: clampOpeningsToWalls() now only calls rackOpeningsOnWall() for a wall if that wall is one of
the two whose length actually changed this drag (tracked via gapBaseline, same as before), or if
it's in genuine overflow (openings no longer fit - a real corruption/safety-net case, kept as-is).
Every other wall's openings are left completely untouched instead of being silently "corrected"
into the wrong spot.

Frontend-only, syntax-checked clean. Worth confirming on a real device: drag a wall that has
plenty of other windows/doors scattered on OTHER walls (especially any not flush against a wall's
far end) and confirm nothing outside the two walls actually being resized moves at all.

## Round 10k - Round 10j's fix wasn't enough - simplified the whole gap model
Nathan retested Round 10j: better, but the corner-slide was still happening on the two walls that
DO legitimately get resized (the ones adjacent to whatever's dragged). Same root cause, just no
longer able to leak onto unrelated walls: rackOpeningsOnWall's proportional gap-scaling model only
ever tracked the space BEFORE each opening on a wall, never the trailing gap after the last one -
so even the walls that ARE supposed to be re-laid-out during a resize would swallow any trailing
gap into the "before" gaps and drag the whole opening group toward the far end. Nathan's call:
"is this the scaling feature? if so, simplify it. this is terrible" - agreed, and rebuilt it.

Old model: freeze each affected wall's gap *ratios* at drag start (computeGapBaseline), then scale
all of them by a single factor as the wall's length changes (rackOpeningsOnWall). Removed entirely,
along with the gapBaseline plumbing through pushWall/startWallDrag - gone completely.

New model (fitOpeningsToWall, replaces rackOpeningsOnWall): an opening's offset is left completely
untouched unless it would actually overlap the previous opening or run past the wall's new
(shorter) end - and even then it's nudged the minimum amount needed, not redistributed. No concept
of "spread the free space out," so there's nothing for it to mismodel. Verified with a standalone
simulation (not just reading the code) covering: wall unchanged, wall grows, wall shrinks but still
fits, wall shrinks past capacity (correctly pulls back from the far end only as far as needed), and
a wall with a large trailing gap after the last opening (previously the exact scenario that broke) -
all came back correct, most as true no-ops.

clampOpeningsToWalls still only touches the two walls whose length actually changed this drag
(now passed explicitly as resizedKeys from pushWall, no baseline object needed) or a wall in
genuine overflow - Round 10j's scoping fix is kept, just simplified alongside everything else.

Frontend-only, syntax-checked clean + logic verified via a standalone Node simulation of the new
function (not full app testing). Worth confirming on a real device: shrink a wall that has several
openings on it down toward its hard-stop and confirm they compress from the correct end only, and
re-run the original repro (drag any wall, confirm nothing on OTHER walls moves, and now also
confirm the two walls that ARE adjacent to the drag behave sensibly instead of jumping).

## Round 10l - four small tweaks - BUILT
1. Service Level (and Glass) dropdowns defaulted to LOOKING like "Rerope"/first-in-list even
   though the stored cell value was still '' - a native <select> with no matching <option> falls
   back to displaying option 0, so it visually looked chosen while exporting blank to PDF. Nathan's
   diagnosis was right. Fixed generically in both dropdown renderers (schedule table + inline
   panel): if the current value isn't found among the real options, a genuine blank option is
   inserted and selected instead of letting the browser guess. Only affects columns that can
   legitimately start unset (Service Level, Glass) - Operation/In Progress already always have a
   real matching value/blank option, so no visible change there.
2. No Work openings no longer show a number badge on the canvas at all - just the greyed-out
   opening shape, no circle+number. If resumed later ("Resume Work"), the badge reappears exactly
   as it was (nothing about the underlying id/number was touched, purely a render-time skip).
3. New fixed "Not in Scope" column added as the last real data column in the Window Schedule,
   right before the "+" add-column button - not part of the preset/custom column system (always
   present, not hideable/reorderable). It's a direct checkbox on o.noWork: checking it is the same
   as tapping "No Work" in focus mode, unchecking it is the same as "Resume Work" - both paths flip
   the identical flag, so they always agree.
4. Added the same delete-confirmation treatment from Round 10g (openings) to the two other focus-
   mode delete buttons that didn't have it yet: "Delete this label" and "Delete this
   rectangle/line". Same danger-styled showConfirmDialog + toast pattern throughout now - all
   three delete entry points in focus mode behave identically.

Frontend-only, syntax-checked clean. Worth confirming on a real device: pick Service Level/Glass
on a fresh window and confirm it now starts genuinely blank (not looking pre-filled); mark a window
No Work and confirm its badge disappears from the canvas immediately, reappears on Resume Work;
toggle the new Not in Scope checkbox in the schedule and confirm it's in sync with the focus-mode
button either direction; try deleting a label and an interior line/rectangle and confirm both now
ask for confirmation first.

## Round 10m - two more tiny UX bugs - FIXED
1. Tapping the Move tool right after a long-press tool menu (e.g. Label's long-press to pick
   Interior Rectangle) closed, needed two taps. Nathan's guess was basically right. Actual
   mechanism: every dialog/menu overlay in this app closes on tapping the backdrop, and that
   listener was on the 'click' event - the overlay sits on top of the whole screen including the
   toolbar, so the first tap's click event only ever reached the overlay (closing it), never the
   Move button underneath. The second tap then landed on the now-revealed real button. Fixed by
   switching every overlay's outside-tap-to-close listener (8 of them, all dialogs/menus in the
   app) from 'click' to 'pointerdown' - removing the overlay that early lets the browser's own
   natural pointerup/click for that SAME gesture fall through and land on whatever's actually
   underneath, so one tap now both dismisses the menu and activates the tool in a single motion.

2. The "x" close button on the focus-mode detail panel didn't actually close it while placing a
   label (and, per Nathan's hunch, the same applied to interior lines/rectangles). Root cause:
   updateInlinePanel() checks selectedInteriorId and selectedLabelId before it ever gets to
   selectedOpeningId - the close button was only ever clearing selectedOpeningId, so with a label
   or interior item selected it just re-rendered the exact same panel right back open. Fixed by
   having the close button call setMode('move') instead of hand-clearing one field - that's the
   same function the Move toolbar button itself calls, so it clears all three selection ids,
   backs out of whatever placement tool was still "armed" (addLabel/addInteriorLine/
   addInteriorRect otherwise stay active for placing several in a row), and exits focus mode back
   to plan view, all in one consistent path.

Frontend-only, syntax-checked clean. Worth confirming on a real device: long-press Label (or
Window) to open its menu, then tap Move once and confirm it activates immediately; place a label,
interior line, and interior rectangle and confirm the panel's "x" fully closes and returns to
plan view for all three, not just openings.

## Round 10n - focus zoom x2, number badge x3 - BUILT
Two constant tweaks per Nathan's numbers:
- FOCUS_ZOOM: 1.3 -> 2.6 (doubled). Well under MAX_ZOOM (6), no clamping.
- Number badge: BADGE_R 15->45, BADGE_FONT 10.5->31.5 (tripled). Also scaled its distance from
  the wall by the same 3x (20->60px, new BADGE_OFFSET constant) so the bigger circle sits in the
  same relative spot instead of swallowing the wall/opening - and bumped the text's vertical
  centering nudge (+4 -> +12) to match the larger font. The separate completed/in-progress/issues
  status badge on the opposite side was NOT touched - Nathan only asked about the number badge.

Frontend-only, syntax-checked clean. Worth a quick look on a real device: badge at 3x is a big
jump - confirm it doesn't crowd neighboring windows on a wall with several close together, and
that focus zoom at 2.6x still leaves enough of the plan visible to get useful context.

## Round 10o - badge size now editable (Options > Badge Size) - BUILT
Round 10n's 3x badges turned out too big. Rather than pick a new fixed number, made it a slider:
new "Badge Size…" item in Options (both crew and full view - legibility on a phone is exactly a
crew concern) opens a dialog with a range input from 15px (the original size) to 45px (Round
10n's 3x). Dragging it live-updates the plan (refreshSvg on input) and saves on release.

Implementation: new state.badgeRadius field (default 25, migrated in applyLoadedState for legacy
plans that predate this), clamped 15-45 via a shared clampBadgeRadius() helper. It's a whole-plan
setting synced through the normal data blob (same mechanism as columnVisibility/
defaultWindowOperation) rather than a per-device preference, so canvas, PDF export, and every
device agree on one size - all three already share the same buildSvg() code path. Font size,
distance from the wall, and the text's vertical-centering nudge all derive from this one number
using the same ratios Nathan already approved at both 15px and 45px (font = radius*0.7, offset =
radius*4/3, text nudge = font*0.381), so the whole badge stays proportionally consistent at any
size in between - not just the circle getting bigger with cramped/oversized text.

Frontend-only, syntax-checked clean. Worth confirming on a real device: drag the slider and watch
the plan update live; close and reopen the plan (or reload) and confirm the chosen size persisted;
export a PDF and confirm it matches the on-screen size.

## Round 10p - French pair leaves share one badge in plan view - BUILT
Discussed fully unifying French pair leaves into one data object (one schedule row, one id) -
real complexity: width-splitting, per-leaf swing/hinge, shared vs. per-leaf photos/issues, and a
migration for existing plans that already have split pairs (some already diverged since today's
leaves are independently editable). Nathan scoped it down to just the canvas badge instead - much
smaller change, no data model touched at all.

Both leaves are still fully independent openings everywhere else - two schedule rows, two ids,
two photo sets, synced independently, no Lambda/migration involved. Only the plan-view badge
changed: the passive leaf now renders no number badge at all, and the active leaf's badge is
positioned at the midpoint of the PAIR's combined span (computed from both leaves' offset/width,
not just its own half) rather than each leaf showing its own badge next to the other. Badge text
also dropped the A/P suffix - was "W12A"/"W12P", now just "W12", once per pair. Falls back to a
leaf's own position/text if its sibling can't be found (shouldn't happen in practice).

Verified the center-point math with a standalone calc (active leaf 180-204, passive 204-228 along
a wall -> badge lands exactly on 204, the shared boundary, as expected) before trusting it.
Frontend-only, syntax-checked clean. Worth confirming on a real device: an existing French pair
(you've got a few - D2, D3 on "Lee Nielsen Plan") should now show one "D2"/"D3" badge centered
between the two leaves instead of two badges side by side; tapping it should still select the
active leaf same as before.

## Round 10q - three feature requests - BUILT
Discussed complexity first (see prior discussion), built in easiest-to-hardest order:

1. **Pulsing issue ring.** Replaced the static glow behind flagged-issue badges with a hollow
   ring that scales outward and fades, repeating - CSS @keyframes, not JS-driven. The one thing
   worth doing right: the whole SVG rebuilds from scratch on nearly every interaction, and a plain
   animation restarts from 0% every rebuild, which would look frozen/stuttery mid-edit. Fixed by
   giving each ring an animation-delay computed from the current wall-clock time (Date.now() %
   PULSE_MS) rather than 0 - a freshly rebuilt ring resumes exactly where it should be, and since
   every flagged opening computes off the same clock, they all stay in phase automatically with no
   coordination code needed. Only the main hasIssues-only glow was replaced - the smaller corner
   accent (issues coexisting with completed/in-progress) stays a static dot, unchanged.

2. **One-finger/mouse pan-drag on empty canvas.** Turned out easier than Nathan expected - hit-
   testing already excludes openings/walls/labels/interior items/joints/handles via a priority-
   ordered chain of data-role checks in onPointerDown, and there was already an unused fallback
   for "nothing matched" (previously just cleared selection). Added startBackgroundPan() there,
   reusing the same viewBox-units-per-screen-pixel technique the existing edge-auto-pan code
   already uses. Important subtlety: deltas are computed from CLIENT-space (physical screen
   pixel) differences against a FIXED gesture-start reference, not from svgPoint()-converted
   differences re-derived every frame - the latter would double-count each frame's own pan into
   the next frame's delta (since the CTM changes as viewState.vx/vy update) and run away. Wires
   into the existing 2-finger-interrupts-1-finger-drag system for free via activeDragCleanup, same
   as every other drag. No-op at zoom 1 (computeViewBox ignores vx/vy until actually zoomed in),
   so it's safe to attach unconditionally.

3. **Bigger focus panel.** Nathan pushed back on his own request mid-discussion - questioned
   whether the "still centered" part needed special handling or would just happen. Talked through
   the SVG math again: the zoomed-in anchor is already set to the exact center of its viewBox on
   both axes, and preserveAspectRatio's default (xMidYMid meet, unchanged) always renders that
   center at the exact center of whatever box the SVG element occupies, regardless of aspect
   ratio - so centering should already be free. Went with just the CSS change (34vh -> 55vh) and
   held off on any container-aware zoom-math rework pending a real-device look, per that
   discussion - if letterboxing turns out to make the content look smaller than it should within
   the strip, that's the follow-up to revisit, not a positioning bug.

Frontend-only, syntax-checked clean. Worth confirming on a real device: flag an issue on two
different windows and confirm both rings pulse in sync; drag on empty background with one finger
(and with a mouse) and confirm the plan pans smoothly without ever grabbing something it shouldn't;
select something in focus mode and confirm it's still centered in the now-smaller visible canvas
strip, and that dragging a window near that strip's edge still auto-pans sensibly.

## Round 10r - field-testing bug report: dropped edits + can't-delete issues - BUILT, NEEDS DEPLOY
Nathan's field-testing report: "we're dropping edits to windows" (Service Level given as the
example) and "I can't delete issues from windows. Once added it sticks around." Two separate root
causes, both diagnosed by re-reading the actual merge/sync code fresh rather than guessing:

1. **Issues couldn't be deleted - by design, not by bug.** `unionIssues`/`mergeKeyedMap` in
   index.mjs union both sides' issue arrays together and nothing more - there was never a
   mechanism to represent "this entry should go away." Clearing `o.issues` locally only ever won
   until the next sync, which would union the old entries right back in from whatever the server
   still had stored.

   Fix: a new `o.issuesClearedAt` timestamp, set client-side the moment "Resolved (clear all)" is
   confirmed (alongside `o.issues = []`). Lambda-side, `mergeKeyedMap` now takes the max of both
   sides' `issuesClearedAt` (new `maxTimestamp` helper) and filters the unioned issues down to only
   entries logged strictly after that mark - so a clear sticks across every future sync, but a
   genuinely new issue logged after the clear (by anyone) still survives normally. Verified with a
   Node simulation covering: clear removes the old entry with zero conflicts; a new issue logged
   after the clear survives a later merge; two people adding different issues concurrently (no
   clear involved) still both survive with zero conflicts, unchanged from before; a real conflict
   on some other field doesn't block the clear from still applying on top. `issuesClearedAt` was
   also added to `openingContentEqual`'s exclusion list (alongside `issues`) so a clear-only change
   is recognized as the non-conflicting case it is, same as an issues-only change already was.

   **Needs Lambda redeploy** - this half of the fix only takes effect once index.mjs is pushed.

2. **Edits silently disappearing - a real race, no second device needed.** Root cause was in
   `applySyncResult` (index.html): once a save's response comes back, it unconditionally
   `Object.assign(o, merged)`s the server's confirmation onto the local opening for anything that
   was part of that request's own payload. That's fine if nothing else happened in the meantime -
   but if the same window got edited AGAIN locally while that first save was still in flight (easy
   to hit solo, just editing fields quickly), the response landing would blindly overwrite that
   newer edit with the older, now-stale server confirmation. No conflict, no toast, nothing to see
   - it just quietly reverted. This is a gap the Round 10d "sent payload" fix didn't cover: 10d
   protects against wrongful *deletion* of things added after a request was sent, but never checked
   for wrongful *overwrite* of things edited again after a request was sent.

   Fix: two new helpers, `deepEqualLocal` (order-independent deep equality, same job as index.mjs's
   own `deepEqual`) and `openingMatchesSent(o, sent)` (same comparison, ignoring `photos` since
   that field is deliberately never sent to or returned by the server). Before `applySyncResult`
   fully applies a response to an opening/label that was part of the just-sent payload, it now
   checks whether the local copy still matches exactly what was sent. If yes (the common case),
   applies normally. If it's diverged - edited again mid-round-trip - only `lastEditedAt` advances
   to the server's confirmed value; the rest of the object is left untouched, preserving the newer
   edit so the next already-queued sync (syncQueuedWhileInFlight already serializes these) carries
   it forward and matches cleanly against the now-correct baseline instead of falsely conflicting.
   Verified with a standalone test of both helpers (identical objects match; diverged objects
   don't; missing sent-record correctly reports no match; key-order-independent equality holds).

   Frontend-only - no Lambda change needed for this half.

Both `index.mjs` and the extracted `<script>` block from `index.html` syntax-check clean
(`node --check`). Worth confirming on a real device once the Lambda is redeployed: log an issue,
clear it, confirm it stays cleared after a reload and after another device/session syncs; edit a
window's Service Level, then immediately edit something else on the same window before the first
save could plausibly finish, and confirm both edits stick after sync settles.

## Round 10s - Batch A: access & data-safety hardening (Lambda) - BUILT, NEEDS DEPLOY + CONFIG
Prompted by an outside code review of index.mjs/index.html. Verified every claim against the
actual code before acting on it (see chat for the full point-by-point check) - all of it held up.
Grouped as one batch since it's all index.mjs and ships in a single redeploy. Nothing here touches
the plan data shape, the merge/conflict logic, or the local-first save architecture - all additive
checks layered on top of the existing design.

1. **Any authenticated Google account could reach every plan.** The Lambda only ever checked that
   an `email` claim existed, not who it belonged to. Fixed with an explicit authorization gate,
   `isAllowedUser()`, checked right after the existing "are you signed in at all" check and before
   any route runs. Configured via two new env vars - `ALLOWED_EMAILS` (comma-separated exact
   addresses) and/or `ALLOWED_DOMAIN` (a single Workspace domain, matched as an exact `@domain`
   suffix, not a loose substring - `@notheartwoodrestore.com` does NOT match `heartwoodrestore.com`,
   verified). **Fails closed**: if neither var is set, every request gets a 403, on purpose - a
   misconfigured deploy should be obvious immediately, not a silent open door. **This means the
   app will reject everyone, including Nathan, until at least one of these two env vars is set on
   the Lambda after this deploys** - see "needs from Nathan" below.

2. **Photo links weren't scoped to their plan.** `getPlan` used to sign a view URL for every key in
   a plan's stored `photos` array with zero check that the key actually belonged to that plan. New
   `photoKeyBelongsToPlan(key, planId)` requires every key to start with `plans/{planId}/openings/`
   before it's ever handed to `getSignedUrl` - anything else (a foreign key, a path-traversal
   attempt) is silently dropped rather than signed. Verified against a same-plan key (passes), a
   different-plan key (blocked), and a `../` traversal attempt (blocked).

3. **Archiving a nonexistent plan created a broken phantom record.** DynamoDB's `UpdateItem`
   creates the item if the key doesn't exist - `archivePlan` had no guard against that, so a typo'd
   or already-gone planId silently created a record with nothing but `planId`/`archived`/edit
   metadata. Added `ConditionExpression: attribute_exists(planId)`; a failed condition now reports
   404 instead of quietly fabricating a record.

4. **Upload-URL requests were unvalidated.** `getUploadUrl` now requires `planId`/`openingId` to
   match a safe identifier pattern (blocks slashes, `..`, and anything else that could escape the
   intended S3 key prefix - not a strict UUID check, since ids are generated in a couple different
   formats client-side, verified against both `nextId()`-style and UUID-style ids), requires `kind`
   to be exactly `"interior"` or `"exterior"`, and now checks the plan actually exists before
   handing out a signed upload slot for it. **Deferred, not built:** genuinely enforcing a max
   upload *byte size* server-side would mean switching from a presigned PUT to a presigned POST
   with a size-range condition, which also changes how the browser performs the upload - bigger
   change than fit this batch, noted for later if it becomes a real problem (an internal crew tool
   uploading phone photos is a low-risk target for this specifically).

5. **No cap on save payload size.** A save larger than roughly 350KB (comfortably under
   DynamoDB's hard 400KB per-item limit) now gets a clean 413 before any parsing or DB work, instead
   of an unbounded body eventually failing unpredictably at the database layer. Checked against the
   raw body size, not the parsed object, so an oversized payload is rejected as cheaply as possible.

6. **Plan list would eventually go silently incomplete.** `listPlans` was a single unpaginated
   `Scan`, which DynamoDB caps at 1MB per call - past that point, plans would just stop appearing
   in the list with no error or indication. Now loops over `LastEvaluatedKey` until the full table
   is read, and uses a `ProjectionExpression` to fetch only the 7 fields the plans screen actually
   displays instead of every full item (including the openings/labels data nobody needed for this
   screen).

7. **Photo URLs signed one at a time.** `getPlan` awaited each `getSignedUrl` call in a loop -
   a plan with two dozen photos paid for two dozen sequential round-trips on every load. Now
   `Promise.all`'d.

8. **Internal error details leaked to the caller.** The catch-all handler returned `err.message`
   straight to the client. Now logs full detail to CloudWatch (prefixed with the request id, so
   it's greppable) and returns a generic "Server error" to the caller. Malformed JSON in a request
   body now cleanly reports 400 instead of falling through to a misleading 500.

All of the above verified with a standalone Node test (stubbed AWS SDK modules, since this sandbox
has no network access to npm) covering: exact-email allow, case-insensitive match, domain match,
domain-suffix-trick rejection, fail-closed-when-unconfigured, same-plan/different-plan/traversal
photo keys, and safe-id accept/reject on both id formats actually used by the frontend. The
existing merge-logic tests were re-run unchanged to confirm nothing in this batch touched that
code path. `node --check` clean on both files.

**Needs from Nathan before this is live:**
- Redeploy index.mjs.
- Set `ALLOWED_EMAILS` and/or `ALLOWED_DOMAIN` on the Lambda's environment variables **before or
  immediately after** that deploy - until one is set, the app is unreachable for everyone (fails
  closed, see above). This is deliberate, but worth knowing going in so it isn't mistaken for a
  bug.

No frontend changes in this batch - index.html is untouched, re-syntax-checked clean regardless.

## Round 10t - Batch B: sync reliability - BUILT, NEEDS DEPLOY + ONE NEW ROUTE
Prompted by a second round of outside review, this time on polling/sync behavior specifically.
Same approach as Round 10s - verified every claim against the real code first (all held up) before
building. Four independent pieces:

1. **"Keep my changes" left per-item baselines stale.** Diagnosed and fixed - see the code comment
   on `handleSaveConflict` for the full mechanism. Short version: that button used to only update
   `state.structuralLastEditedAt`, never the individual opening/label baselines that the same
   conflicted save had already gotten fresh values for server-side (the per-item merge always runs
   alongside a structural conflict, per computeSave). Left stale, those could cause a false "someone
   else changed this too" conflict on a later retry, on an opening nobody actually double-touched.
   Fixed by routing "Keep my changes" through `applySyncResult` - the same reconciliation a normal
   200 already uses - instead of hand-rolling a partial version of it. Verified with two simulated
   scenarios: baselines advance correctly and content is preserved on a normal resolve, and the
   Round 10r divergence protection (an opening edited again while the request was in flight) still
   holds on this path too.

2. **No automatic retry after a real failure.** A failed save (network error, non-2xx, not a 409 -
   409 already correctly routes to the conflict dialog instead) used to just sit there until another
   edit happened or someone tapped the indicator. Now retries automatically at roughly 5s/15s/30s/
   60s/120s, holding at 120s for as long as failures continue, each with +/-20% random jitter so
   multiple crew devices losing signal together don't all retry in the same instant. Skips arming a
   timer if a fresh edit already queued its own immediate retry (no point running both). Cleared on
   genuine success or when connectivity actually returns (see below) rather than waiting out
   whatever's left of the countdown.

3. **No awareness of the browser going on/offline.** Added `online`/`offline` listeners - offline
   cancels any pending retry timer (nothing to retry against), online immediately retries a stuck
   save instead of waiting out the backoff. The app coming back into view (tab switch, phone wake)
   also now triggers an immediate retry attempt if there's unflushed work and the browser thinks
   it's online, on top of the existing "check for a newer version" that already happened here.

4. **The 5-minute background poll pulled the entire plan.** `checkForNewerVersion` used to call the
   same full `GET /plans/{planId}` a real plan load uses - every field, plus a freshly signed URL
   for every photo - just to compare one timestamp. New `GET /plans/{planId}/revision` route
   (Lambda: `getPlanRevision`) returns only `planId`/`lastEditedAt`/`structuralLastEditedAt`/
   `lastEditedBy` via a `ProjectionExpression`, so a routine "did anything change" check is cheap.
   The full plan is only fetched afterward, and only when the answer is yes. **Degrades safely if
   the new route isn't live yet**: a non-ok response (including a 404 before the route exists) falls
   back to the exact old full-plan check, so nothing regresses in the meantime - it'll just make two
   requests per poll instead of one until the route is added, not fail outright.

Also worth noting: the reviewer's suggestion to "never discard the IndexedDB copy until the server
confirms" is already true today (local save happens first, unconditionally, on every edit,
regardless of cloud outcome) - no change needed there, just confirmed.

`node --check` clean on both files. Retry/backoff logic isn't independently unit-testable the way
the merge logic is (it's timer-driven, not pure), so it got a careful read-through instead - flagged
as one of the things worth specifically watching on a real device (see checklist below).

**Needs from Nathan:**
- Redeploy index.mjs (same deploy as anything else pending - this doesn't need its own).
- Add a new API Gateway route, `GET /plans/{planId}/revision`, pointing at the same Lambda with the
  same JWT authorizer as the existing `GET /plans/{planId}` route - same steps you used to set that
  one up originally. Not required for anything else in this batch to work; only the poll-efficiency
  piece depends on it, and it degrades gracefully without it.

**Worth confirming on a real device:** put the phone in airplane mode mid-edit, confirm the retry
countdown kicks in when you reconnect (should fire immediately, not wait out the timer); force a
structural conflict between two devices, choose "keep mine," and confirm a subsequent edit to an
untouched window doesn't report a spurious conflict; leave the tab backgrounded for a few minutes
with an unsynced edit, then bring it back to front and confirm it retries right away instead of
waiting for the next 5-minute tick.

## Round 10u - Batch D: issue log UI + per-issue delete - BUILT, NEEDS DEPLOY
Two requests: reorder the issue log entry layout (message above who/when, not below), and add a
per-issue "x" so one issue can be deleted at a time instead of only all-or-nothing.

1. **Layout.** Each entry's message now renders above its byline. Small CSS/DOM reorder only.

2. **Per-issue delete.** The all-or-nothing "Resolved (clear all)" from Round 10r used a single
   timestamp watermark (`issuesClearedAt`) that only knew how to say "everything before now is
   gone" - it had no way to name one specific entry. Since that mechanism was built but never
   actually deployed, swapped it out entirely rather than running two systems side by side (Nathan
   signed off on this before it was built):
   - Every new issue now gets a stable `id` at creation (`nextId()`), alongside the existing
     `time`/`user`/`message`.
   - A new `deletedIssueIds` array on each opening is a grow-only tombstone set - same pattern
     already used for deleted openings/labels elsewhere in this app, just scoped to individual
     issue entries. Deleting an issue (the new x button, or "Resolved" clearing all of them one by
     one under the hood) adds its identity to that list; the Lambda unions both sides'
     `deletedIssueIds` and filters anything in that set out of the unioned issues.
   - **Old issues already in production, logged before this shipped, never got an id.** Rather than
     needing a migration pass, both the Lambda's `issueKey()` and the frontend's matching
     `issueKeyLocal()` fall back to the same `(time, user, message)` tuple `unionIssues` has always
     deduped by whenever `id` is missing - so a pre-existing issue is just as deletable as a new
     one, verified directly (see below).
   - `openingContentEqual`'s exclusion list swapped from `["issues", "issuesClearedAt"]` to
     `["issues", "deletedIssueIds"]` so a delete-only change still counts as the "never a conflict"
     case, same as an issues-only change already did.

Verified with five Node scenarios against the real merge function: delete one issue by id (the
other survives, zero conflicts), clear-all via the same mechanism, a new issue logged after a
clear still survives a later merge, a legacy no-id issue deleted through its fallback key, and two
people deleting *different* issues concurrently (both stick, zero conflicts - this one matters
because it's the exact situation the old all-or-nothing button couldn't represent at all). Also
directly confirmed the frontend's `issueKeyLocal` and the Lambda's `issueKey` produce identical
output for both an id-based entry and a legacy no-id entry - if these two ever drift apart, a
delete computed client-side wouldn't match what the server is filtering against, so this was worth
checking explicitly rather than assuming. `node --check` clean on both files.

**Needs from Nathan:** redeploy index.mjs (same deploy as anything else pending).

## Round 10v - Batch C: sync status UX polish - BUILT, frontend only
Last piece of the queue from the two outside reviews - no correctness bug here, just making the
existing states easier to read at a glance, per the reviewer's four-state framing.

1. **Friendlier, more specific wording.** "Syncing…" -> "Saving to team…", "Synced" -> "Saved to
   team" (adds "- review needed" when a merge conflict needs a look, same as before), "Conflict -
   action needed" -> "Conflict - choose a version".

2. **Elapsed-time framing when work hasn't reached the team.** New `updateCloudStatusText()`
   computes "Saved on this device - last synced 3m ago" (or "Offline - saved on this device - ..."
   when `navigator.onLine` is false, or "not yet synced this session" if it never has been) any time
   there's unsynced work and nothing else is actively happening. Two triggers: immediately after a
   failed save (so this shows up right away, not after the old 10-minute wait for the big banner),
   and on the existing 30-second banner tick (so "3m ago" doesn't go stale if nobody touches
   anything for a while). Guarded against overwriting "Saving to team…" or "Conflict - choose a
   version" while either of those is the real current state (`syncInFlight` / new `conflictPending`
   flag, set true when the conflict dialog opens and cleared when either option is chosen).

3. **Close/navigate-away warning.** New `beforeunload` listener prompts if there's unsynced work
   when closing the tab or navigating away. Browsers ignore custom message text in this dialog for
   security reasons and show their own generic wording - `returnValue` still has to be set to
   trigger the prompt at all, which is what's actually happening here. Plans are opened via a real
   page navigation in this app (not an in-page swap), so this also covers "switching plans" with
   unsynced work, not just closing the tab entirely.

The big 10-minute banner and its wording are unchanged - it's still the "you've really been out of
touch a while" escalation; the small indicator now carries the immediate signal the reviewer asked
for. Verified `formatElapsed`/`updateCloudStatusText` directly (six cases: no-op when already
synced, never-synced-this-session wording, elapsed-time wording, offline prefix, and both guard
flags correctly blocking an overwrite). `node --check` clean.

## This session's full queue, start to finish
Two rounds of outside code review (access/data-safety, then sync/polling specifically) plus one
UI request, worked through as four batches - Round 10s (Batch A), 10t (Batch B), 10u (Batch D),
10v (Batch C) above, in that build order. Every review claim was verified against the real code
before anything was built on top of it - see each round's section for the specific checks run.

**Still needed from Nathan, all in one pass:**
1. Redeploy `index.mjs` (covers Batches A, B, and D - one deploy).
2. Set `ALLOWED_DOMAIN=heartwoodrestore.com` on the Lambda's environment variables (Batch A -
   the app rejects everyone until this is set, see Round 10s).
3. Add a new API Gateway route, `GET /plans/{planId}/revision`, pointing at the same Lambda with
   the same JWT authorizer as the existing `GET /plans/{planId}` route (Batch B's poll-efficiency
   piece only - everything else in this queue works without it, and this one degrades gracefully
   if skipped for now).

No frontend deploy step needed beyond whatever Nathan's normal process is for index.html.

## Round 10w - six small tweaks - BUILT, frontend only
Confirmed the two visual ones (DH, Slider) with Nathan via mockups before touching the rendering
code, since prose descriptions of plan-view symbols are easy to build the wrong interpretation of.

1. **Double Hung - stacked, not side-by-side.** Old symbol was two lines offset +/-4 from the wall
   centerline plus a perpendicular tick at the midpoint (the meeting rail) - visually that tick
   read as a seam splitting the window into two side-by-side halves. New symbol: three full-width
   lines, evenly spaced, centered on the wall, no center tick. The opening's whole drawn depth
   (`half`, which the cutout polygon and jamb end ticks both already read from) doubles specifically
   for Double Hung windows so the three lines have room - confirmed this stays centered and the
   outer lines stay safely inside the jamb ticks with a Node check of the actual geometry.

2. **New window type: Slider.** Added to `WINDOW_OPERATIONS`, the default-window-operation picker
   (badge abbreviation "SLD"), and gets its own plan symbol: two rectangle outlines, one from the
   start jamb to the opening's true midpoint, the other from the midpoint to the end jamb, each
   touching the wall centerline at that shared midpoint (verified directly - both rectangles'
   point lists contain the exact same corner coordinate) so together they read as one continuous
   line straight across, with one sash "in" and one "out" above/below it. Same depth as every
   other window type except Double Hung - no doubling here.

3. **Flip Orientation button.** New `o.sliderFlipped` boolean, toggled from a button in the inline
   panel (shown only for Slider windows, in the same slot the door/casement swing controls already
   use). Flipping just negates which side each rectangle's outer edge points toward - verified the
   flipped geometry swaps sides correctly while the shared centerline corner stays put.

4. **Two new Service Level options**: "New Sash" and "New Unit (Frame and Sash)", added to
   `SERVICE_LEVEL_OPTIONS` ahead of "Other" (kept last, matching every other options list in the
   app).

5. **New "Service Description" column**, a plain text field, added to `presetColumnDefs()`
   immediately after Service Level. No special-case code needed - schedule cells, the inline
   panel, and CSV export all read columns generically off `col.key`, so a new text column just
   works once it's in the list.

6. **Not in Scope moved to the far right** of the schedule table, now to the right of the Add
   Column button instead of to its left - header and row cells reordered together, column count
   unchanged so nothing else shifts.

7. **Completed rows get a light green tint.** New `.rowCompleted` rule, scoped with `#sched` to
   outrank the existing `td[contenteditable]` background rule (same specificity, later in the
   stylesheet wins) so it actually shows through on every cell in the row, not just the
   non-editable ones. Not-in-scope's gray tint still wins if a row is somehow both (its rule uses
   `!important`), which is the right call.

Frontend only, `node --check` clean. Worth a look on a real device: add a Double Hung and a Slider
side by side and confirm they read clearly at normal zoom (not just close up); flip a slider a
couple times and confirm the drawing updates immediately; mark a row complete and confirm the tint
shows even in edited text cells, not just the checkbox cell.

## Round 10x - three fixes from field feedback on Round 10w - BUILT, frontend only

1. **Double Hung jamb edges poking past the wall face.** Round 10w doubled `half` for Double Hung
   windows to give the new 3-line symbol room, but `half` is also what the cutout polygon and the
   jamb end ticks (the short black lines marking the wall's actual interior/exterior faces) are
   built from - so doubling it pushed those past the true wall face too, which looked wrong (every
   other window type's ticks correctly stop right at the face). Fixed by leaving `half` alone for
   every opening type again - the "stacked" look now comes entirely from the 3-line spacing fitting
   within the same footprint every other window uses, not from widening the opening itself. The
   line offsets (`half*0.55`) recompute automatically off the now-unchanged `half`, landing close to
   the original two-line spread (~4 units either way, same as before).

2. **Issue log byline.** Three changes to the meta line under each issue's message: dropped its
   font-weight from bold to normal (it was already `var(--muted)` gray, but bold gray at that size
   was still pulling the eye ahead of the message itself); a new `shortIssueUser()` strips a
   trailing `@heartwoodrestore.com` case-insensitively for display only (the full address is
   untouched in storage - this is purely how it's shown); a new `shortIssueTime()` formats with
   `toLocaleTimeString([], {hour:'2-digit', minute:'2-digit'})` instead of the default
   `toLocaleString()`, which drops the seconds. Verified both helpers directly (domain strips
   case-insensitively, a non-Heartwood email is left alone, missing values fall back to
   "Unknown"/empty string without crashing, and the time output has no seconds component).

3. **Completed-row green tint wasn't showing.** Root cause: Round 10w added the CSS rule
   (`.rowCompleted`) but never actually added the `rowCompleted` class to the row - the styling was
   real, nothing was ever there to apply it to. One-line fix to the `tr.className` assignment in
   `buildSchedule`, checking `o.cells.completed`. Verified directly against a few opening shapes,
   including one with no `cells` at all, to confirm it doesn't throw.

`node --check` clean. Frontend only - no Lambda involved in any of these three.

## Round 10y - DH geometry actually fixed this time + issue byline lighter still - BUILT
Round 10x's "fix" over-corrected: reverting `half` fixed the jamb-tick horns but also crushed the
3 sash lines back together, since their spacing was computed off that same `half` - two separate
problems that had gotten tangled into one shared variable.

Split them apart. `half` (jamb end ticks, and the cutout for every window type except Double Hung)
never changes again - that's what keeps the black jamb marks exactly where every other window's
already sit, right at the true wall face. A new `cutoutHalf` is Double Hung-only and wider (double
`half`) - it drives both the white paper cutout AND the 3-line spacing, decoupled entirely from the
jamb ticks. Widening the cutout specifically is safe to do because it's white polygon on white page
background (`--paper` is `#ffffff` both times) - it's invisible past the true wall edge regardless,
unlike the black jamb ticks, which very visibly are not the background. Verified the actual numbers:
jamb ticks stay at 7.25 (unchanged, matches every other window type), the cutout widens to 14.5 for
Double Hung, and the 3 sash lines land at roughly +/-8 - comfortably inside that wider cutout, with
real gaps between all three lines again.

Also lightened the issue log byline further - it was already non-bold `var(--muted)` (#4b5058) from
Round 10x, but that still read as fairly dark next to the message. Now a dedicated lighter gray
(#9aa0a6) just for this one line, rather than further lightening `--muted` itself, which other UI
(empty-state text, etc.) still relies on at its current darker weight.

`node --check` clean. Frontend only.

## Round 10z - In-progress badge recolored to amber with opacity gradient - BUILT
The orange "in progress %" badge read too close to the red issues badge - deceiving at a glance.
Discussed options with Nathan via mockups (color choices, then a black-vs-white text follow-up)
before touching code, per the usual pattern for ambiguous visual changes. Landed on: Amber
(#f2a900), black text throughout, and the badge gets more solid as progress increases rather than
staying flat.

New `--inprogress:#f2a900` CSS variable added in `:root` (and mirrored into the duplicate
print/export style string, since that one doesn't inherit from the page stylesheet). Badge fill
now reads `var(--inprogress)` instead of `var(--accent)` for the in-progress case only - `completed`
(green check) and issues-only (white dots) badges are untouched.

Opacity gradient via a small lookup, `fill-opacity` on the status circle: 25%->0.4, 50%->0.6,
75%->0.8, 90%->1 (full color). Percentage text switched from white to `var(--ink)` (near-black) for
the in-progress case specifically - Nathan confirmed black throughout rather than switching to white
at higher opacity/percentage steps, overriding my legibility concern.

Verified the opacity lookup and color/text branching in isolation via Node (all four percentages
map correctly, completed/issues-only paths unaffected). `node --check` clean on the extracted
script block. Frontend only - no Lambda changes.

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
