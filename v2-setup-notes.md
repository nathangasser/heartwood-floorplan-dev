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
- Real-device testing (task 9) - not yet run
