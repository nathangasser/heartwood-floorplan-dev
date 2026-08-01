# Floor Plan App V2 — Setup Notes

Running record of config values created during AWS/Google setup. Reference this when writing frontend/Lambda code.

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
- Backlog, roughly in priority order: Crew Mode (full design agreed - default-crew share links,
  per-device remembered preference via localStorage, lands on Plan tab, no drawing toolbar,
  trimmed header, "Switch to Full View" in a menu, "Switch to Crew View" added to Options for
  previewing - not yet built), per-opening color tag/flag/comment, Make.com/Airtable
  auto-archive webhook (needs airtableRecordId field + a machine-auth endpoint), then promote
  beta to production once everything's solid.
