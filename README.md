# coe-portal

The external staff submission form for the Office of the Controller of Examinations. External examiners and staff use it to submit their claim details (name, work type, bank details, and date rows for hall duty or practicals) without needing a login.

It is a single file, `index.html` (React and Babel in the browser, Supabase JS client). It shares one Supabase project with the admin app, [coe-app](../coe-app).

## What it does

- Visitors open a link from the COE office and complete a 4-step form: work type, personal details, bank details (IFSC is checked live), review and submit.
- Each submission becomes a **draft** claim (`status = 'draft'`, `entry_source = 'portal'`). Nothing is paid or final until an admin approves it in coe-app.
- Visitors get a reference number (the first 8 characters of the claim id) to quote to the COE office.

## How the `?t=` token works

The admin app generates a link like:

```
https://<portal host>/?t=<token>
```

where `<token>` is `base64url( SEASONID + "_" + EXPIRY_MS )`, for example the season `ND25` expiring at a given millisecond timestamp.

On load, and every second, the portal decodes the token and:

- no token, or a malformed one: shows "Submissions Closed",
- expiry in the past: shows "Link Expired",
- otherwise: shows the form for that season and loads the season row from `exam_seasons`.

**This check runs only in the visitor's browser.** The token is not signed or secret, so anyone can change the season or expiry in it. It only keeps honest visitors from using stale links. What actually decides whether a submission is accepted is the database policy, not this page. See the policies in [`coe-app/db/rls_policies.sql`](../coe-app/db/rls_policies.sql) and the security notes in the [coe-app README](../coe-app/README.md#security-notes).

Note: if the token is missing or unreadable, the code falls back to the hard-coded season id `ND25` for the season lookup (the form itself stays closed).

## Database access

The portal uses the public (publishable) key as the `anon` role. With the policies in `coe-app/db/`, it can only:

- read `exam_seasons`,
- insert into `claims` (as a draft) and `claim_date_rows`, and only while the season's `portal_access` row is active and unexpired.

Set up the tables and policies from [`coe-app/db/`](../coe-app/db/) (`schema.sql`, then `rls_policies.sql`, then `audit_triggers.sql`).

## Configuration

`SUPA_URL` and `SUPA_KEY` are set near the top of the script in `index.html`. The key is a publishable key and is meant to be public. Never put a `service_role` key here.

## Running it

Open `index.html` through any static host. It needs a valid `?t=` link to show the form.
