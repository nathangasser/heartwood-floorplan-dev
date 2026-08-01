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
