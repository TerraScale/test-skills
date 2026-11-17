# Export, transform and import Firebase users

This reference moves the Firebase user directory into a TerraScale project and
builds the mapping table. It never moves a password: TerraScale accepts only
bcrypt, Argon2id PHC and Django PBKDF2-SHA256 hashes on import, and Firebase uses
its own modified scrypt
([user profiles and imports](https://github.com/TerraScale/terrascale/blob/main/docs/auth/user-profiles.md#import),
[Firebase CLI auth reference](https://firebase.google.com/docs/cli/auth)).

## PII and secret handling

The export contains emails, phone numbers, names, photo URLs, provider ids and,
for scrypt accounts, password hashes and salts. Treat it like a database dump.

- Work in an encrypted, access-controlled location outside every repository.
  Add the file names to the global ignore list before you export.
- Never paste rows into tickets, chats or AI prompts. Logs show counts and uids,
  never emails or hashes.
- The password hash parameters (base64 signer key, base64 salt separator, rounds,
  memory cost) are shown in the Firebase console under Authentication > Users >
  the menu above the user list, and Firebase marks them sensitive. TerraScale does
  not use them. Copy them only if you must keep a rollback path into another
  Firebase or Identity Platform project, and then only into a secret manager as
  `FIREBASE_SCRYPT_SIGNER_KEY`, `FIREBASE_SCRYPT_SALT_SEPARATOR`,
  `FIREBASE_SCRYPT_ROUNDS` and `FIREBASE_SCRYPT_MEM_COST`. Never echo them, pass
  them as visible command arguments in shared terminals, or commit them.
- Strip `passwordHash` and `salt` from every derived file. Only the original
  export keeps them, for rollback.
- Delete the export, the derived files and any copied hash parameters on the
  retention date in the migration plan, and record the deletion.
- TerraScale's directory export is itself audited as `project.users_exported` per
  page ([export the directory](https://docs.terrascale.tech/guides/end-user-authentication#export-the-directory)).

## 1. Export from Firebase

~~~sh
firebase login
firebase auth:export users.json --format=json --project "$FIREBASE_PROJECT_ID"
~~~

`auth:export ACCOUNT_FILE --format=json|csv` is documented in the
[auth reference](https://firebase.google.com/docs/cli/auth); `--project` is the
CLI's global project option ([Firebase CLI](https://firebase.google.com/docs/cli)).
The documented record fields are `localId`, `email`, `emailVerified`,
`passwordHash`, `salt`, `displayName`, `photoUrl`, `createdAt`, `lastSignedInAt`,
`phoneNumber` and `providerUserInfo` (`providerId`, `rawId`, `email`,
`displayName`, `photoUrl`). Only scrypt accounts have a hash.

The reference does not list custom claims, MFA enrollments, the disabled flag or
tenant users, so read those with the Admin SDK `listUsers`
([list all users](https://firebase.google.com/docs/auth/admin/manage-users#list_all_users)).
Run this on a trusted machine with `GOOGLE_APPLICATION_CREDENTIALS` pointing at a
service account key, and set `FIREBASE_TENANT_ID` for one Identity Platform tenant:

~~~js
import { writeFileSync } from "node:fs";
import { initializeApp, applicationDefault } from "firebase-admin/app";
import { getAuth } from "firebase-admin/auth";

initializeApp({ credential: applicationDefault() });
const tenant = process.env.FIREBASE_TENANT_ID;
const auth = tenant ? getAuth().tenantManager().authForTenant(tenant) : getAuth();

const extras = {};
let pageToken;
do {
  const page = await auth.listUsers(1000, pageToken);
  for (const user of page.users) {
    extras[user.uid] = {
      disabled: user.disabled,
      customClaims: user.customClaims ?? null,
      mfa: (user.multiFactor?.enrolledFactors ?? []).map((factor) => factor.factorId),
    };
  }
  pageToken = page.pageToken;
} while (pageToken);

writeFileSync("extras.json", JSON.stringify(extras));
console.log(`users: ${Object.keys(extras).length}`);
~~~

For multi-tenant projects, export each tenant with this script (setting
`FIREBASE_TENANT_ID`) and keep each tenant's files apart.

## 2. Freeze and re-export

The first export is for rehearsal. On cutover day, block new Firebase accounts
with a `beforeUserCreated` blocking function
([blocking functions](https://firebase.google.com/docs/auth/extend-with-blocking-functions)),
export again, and import only uids not imported before.

## 3. Transform

The transform writes three files: import batches without hashes, the mapping
skeleton with one new `app_user_id` (a ULID) per Firebase uid, and a skip list
with a reason for every user that is not imported. Save it as
`transform.mjs` outside the repository and run
`node transform.mjs users.json extras.json out`.

~~~js
import { readFileSync, writeFileSync, mkdirSync } from "node:fs";
import { randomBytes } from "node:crypto";

const [exportFile, extrasFile, outDir] = process.argv.slice(2);
const users = JSON.parse(readFileSync(exportFile, "utf8")).users;
const extras = JSON.parse(readFileSync(extrasFile, "utf8"));
mkdirSync(outDir, { recursive: true });

const alphabet = "0123456789ABCDEFGHJKMNPQRSTVWXYZ";
function ulid() {
  let time = Date.now();
  let head = "";
  for (let i = 0; i < 10; i++) {
    head = alphabet[time % 32] + head;
    time = Math.floor(time / 32);
  }
  const bytes = randomBytes(16);
  let tail = "";
  for (let i = 0; i < 16; i++) tail += alphabet[bytes[i] % 32];
  return head + tail;
}

function httpsUrl(value) {
  return typeof value === "string" && value.startsWith("https://") && value.length <= 2048 ? value : undefined;
}

const byEmail = new Map();
for (const user of users) {
  if (!user.email) continue;
  const key = user.email.trim().toLowerCase();
  byEmail.set(key, [...(byEmail.get(key) ?? []), user]);
}

function survivor(group) {
  return [...group].sort((a, b) =>
    Number(b.emailVerified === true) - Number(a.emailVerified === true)
    || Number(b.lastSignedInAt ?? 0) - Number(a.lastSignedInAt ?? 0))[0];
}

const rows = [];
const mapping = [];
const skipped = [];
const appIds = new Map();

for (const [email, group] of byEmail) {
  const keep = survivor(group);
  const appUserId = ulid();
  for (const user of group) {
    appIds.set(user.localId, appUserId);
    mapping.push({ firebase_uid: user.localId, app_user_id: appUserId, email_key: email, survivor_uid: keep.localId });
  }
  const extra = extras[keep.localId] ?? {};
  rows.push({
    email: keep.email.trim(),
    email_verified: keep.emailVerified === true,
    external_id: keep.localId,
    first_name: keep.displayName ? keep.displayName.slice(0, 255) : undefined,
    image_url: httpsUrl(keep.photoUrl),
    private_metadata: {
      firebase_providers: (keep.providerUserInfo ?? []).map((info) => info.providerId),
      firebase_mfa: (extra.mfa ?? []).length > 0,
      firebase_disabled: extra.disabled === true,
      firebase_merged_uids: group.filter((user) => user !== keep).map((user) => user.localId),
    },
  });
}

for (const user of users) {
  if (user.email) continue;
  const appUserId = ulid();
  appIds.set(user.localId, appUserId);
  mapping.push({ firebase_uid: user.localId, app_user_id: appUserId, email_key: null, survivor_uid: null });
  skipped.push({ firebase_uid: user.localId, reason: user.phoneNumber ? "phone_only" : "no_email" });
}

for (let i = 0; i < rows.length; i += 500) {
  writeFileSync(`${outDir}/batch-${String(i / 500).padStart(4, "0")}.json`, JSON.stringify({ users: rows.slice(i, i + 500) }));
}
writeFileSync(`${outDir}/mapping.json`, JSON.stringify(mapping));
writeFileSync(`${outDir}/skipped.json`, JSON.stringify(skipped));
console.log(`rows: ${rows.length} mapped: ${mapping.length} skipped: ${skipped.length} merged: ${mapping.length - rows.length - skipped.length}`);
~~~

Review before importing:

- **Duplicates.** The survivor is the verified account, then the most recent
  sign-in. Every merged uid points at the survivor's `app_user_id`. Check the
  merges by hand when a group mixes verified and unverified accounts: an
  unverified duplicate may belong to someone else, so do not merge its data until
  the surviving user has proven the mailbox.
- **Email case.** The grouping key lowercases emails so case variants merge;
  TerraScale refuses a duplicate email with `user_conflict`.
- **Names.** `displayName` goes to `first_name` whole; split it only if your app
  already stores given and family names.
- **Custom claims** stay out of the batches; they are set after import.
- **No hashes.** The rows contain no `password_hash`. Search the output folder for
  `passwordHash` and `salt` before you continue; both must be absent.

## 4. Import into TerraScale

`POST /v1/{project}/users/import` takes 1–500 rows, commits each valid row
independently, sends no email, and allows ten calls per project per minute. It
needs a project admin's sign-in token or a project secret API key with
`auth:write`
([user profiles and imports](https://github.com/TerraScale/terrascale/blob/main/docs/auth/user-profiles.md#import)).
Keep the token in `TERRASCALE_ACCESS_TOKEN`, read from a secret manager.

~~~sh
api() { curl -sS --fail-with-body -H @<(printf 'Authorization: Bearer %s\n' "$TERRASCALE_ACCESS_TOKEN") "$@"; }
for batch in out/batch-*.json; do
  result="${batch%.json}.result.json"
  [ -s "$result" ] && continue
  api -X POST "https://auth.terrascale.tech/v1/$TERRASCALE_PROJECT_ID/users/import" \
    -H "Content-Type: application/json" --data-binary @"$batch" > "$result"
  jq -r --arg b "$batch" '"\($b) imported=\(.items|length) errors=\(.errors|length)"' "$result"
  sleep 7
done
~~~

The response is `{"items":[{"index":0,"user":{...}}],"errors":[{"index":1,"code":"...","message":"..."}]}`
with zero-based row indexes. For each batch:

- Write `items[].user.id` as `terrascale_user_id` into the mapping rows whose
  `firebase_uid` equals `items[].user.external_id`, including merged uids.
- `invalid_request` errors: fix the row (for example a non-HTTPS photo URL or a
  control character in a name) and import it in a new batch.
- `user_conflict` errors: the email or `external_id` already exists. Read the
  existing user before resubmitting. A lost response can leave committed rows, so
  a retried batch reports them as conflicts; reconcile them by email instead of
  creating anything new.
- A `429` means the per-minute limit; wait and resume. The loop skips batches that
  already have a result file.

## 5. Claims and disabled users

Set custom claims per user with `PATCH /v1/{project}/users/{user}` and a
`custom_claims` object that fits the limits in
[feature mapping](feature-mapping.md#custom-claims):

~~~sh
api -X PATCH "https://auth.terrascale.tech/v1/$TERRASCALE_PROJECT_ID/users/$TERRASCALE_USER_ID" \
  -H "Content-Type: application/json" -d '{"custom_claims":{"is_admin":true,"plan":"pro"}}'
~~~

Disable every user whose `private_metadata.firebase_disabled` is true in the
dashboard or with the SDK project client's disable call
([manage users](https://docs.terrascale.tech/guides/end-user-authentication#manage-users)).
Import has no disabled field, so do this before you open sign-in.

## 6. Reconcile

Export the TerraScale directory page by page
([export the directory](https://docs.terrascale.tech/guides/end-user-authentication#export-the-directory)):

~~~sh
api -G "https://auth.terrascale.tech/v1/$TERRASCALE_PROJECT_ID/users/export" --data-urlencode limit=100 > ts-page-1.json
~~~

Repeat with `after` set to the previous `next` until the response has no
`X-Export-Next` header, saving each page and deduplicating by `id`. Then check:

- Every import row has exactly one TerraScale user with the same `email`, and the
  mapping row of its uid has that user's `id`.
- `email_verified` matches Firebase for every imported user. Repository source
  shows the export returns the stored value, which is `false` for imported
  unverified users, although the public page still says it is always `true`.
- `disabled` is true exactly for the users disabled in Firebase.
- No TerraScale user lacks an `external_id` unless it signed up after cutover.
- Every Firebase uid is in `mapping.json`, and every uid without a
  `terrascale_user_id` is in `skipped.json` with a reason.

The export is a creation-cutoff scan, not a snapshot: pause sign-up and imports
while you reconcile.
