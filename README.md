# ValidApp V2

This V2 build upgrades the original single-page ValidApp repository into a role-based validation workflow with:

- Creator / Viewer / Reviewer roles, plus Section Head, Deputy QA Manager, QA Manager and Administrator for the approval chain.
- Creator-only study creation and document upload.
- PDF / DOC / DOCX / XLS / XLSX upload to Firebase Storage (25 MB per file).
- Sequential workflow: Draft → Section Head → Reviewer → Deputy QA → QA Manager → Approved.
- Any approval-stage reviewer can return the study to the Creator with a mandatory comment.
- Creator can revise/resubmit after return.
- Review history stored as separate immutable records.
- Append-only audit log for login/logout, viewing, study creation/edit/save/submit/resubmit, uploads/downloads, comments, returns, status changes, approvals, rejections, user creation and audit/user directory access.
- Firestore and Storage rules intended to enforce role permissions server-side.

## Firebase setup

1. In Firebase Console for project `validapp-db`, enable **Authentication → Sign-in method → Email/Password**.
2. Publish `firestore.rules` and `storage.rules` from this folder.
3. Create the first Administrator in Firebase Authentication.
4. Create that user's ValidApp profile document at:
   `artifacts/validapp-default/public/data/users/<AUTH_UID>`
   with fields:
   - `uid`: Auth UID
   - `email`: account email
   - `displayName`: name
   - `role`: `Administrator`
   - `department`: `QA`
   - `active`: `true`
5. Sign in. Use **Users** to create additional accounts. The browser flow generates a temporary password; use Firebase's password-reset flow for a permanent password.

## Important migration note

The original repository used a Firestore username/password document and anonymous authentication. V2 intentionally removes that insecure login pattern and uses Firebase Authentication. Existing study data under the same `records` collection remains readable if it already matches the V2 path. Existing users will need to be migrated into Firebase Authentication and new role profile documents.

## GitHub Pages

Replace the repository's existing `index.html` with the V2 `index.html`, publish the rules to Firebase, commit, and allow GitHub Pages to redeploy.
