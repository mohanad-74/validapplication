# ValidApp Enhanced — Controlled Validation / Quality Workflow

## What was preserved
The enhancement keeps the original ValidApp Firebase architecture and existing `artifacts/{appId}/public/data` pattern. The original study categories and core fields remain available. Existing records are not deleted.

The uploaded source showed a legacy local-password user model, anonymous Firebase authentication, client-side role checks, plaintext passwords in Firestore, simple `records/users/audit` collections, and a single generic review status. These are security and data-integrity weaknesses for a controlled quality application. The enhanced build moves authentication to Firebase Authentication, uses UID-based user profiles, adds backend-enforced role checks, immutable audit writes, controlled file storage, and explicit workflow states.

## Important migration point
The legacy code stored passwords in Firestore and identified users by `username`. Passwords must **not** be migrated into the new system. Existing Firebase Authentication accounts can be reused if they already exist. Each account needs a `users/{auth.uid}` profile containing its role.

For old Firestore user records, preserve the old documents. Create UID-based profile documents for the corresponding Firebase Authentication accounts. The application also retains a limited legacy-owner compatibility check using the old `createdBy` value so old study ownership can continue to be recognized where the profile username matches.

### Recommended UID profile
```text
users/{firebaseAuthUid}
  email: "user@company.com"
  displayName: "User Name"
  username: "legacyUsername"       // optional compatibility field
  role: "Creator"
  department: "Validation"
  disabled: false
  createdAt: server timestamp
  createdByUid: "admin uid"
```

## Collections
Keep the existing path:
```text
artifacts/{appId}/public/data/
```
Collections used by the enhanced application:
```text
users
records                 // existing validation studies
studyFiles              // metadata only; binary files remain in Storage
reviews                 // permanent review/comment history
calibration
qualification
audit                   // append-only audit records
```

## Storage path
```text
artifacts/{appId}/study/{studyId}/{fileId}_{safeFileName}
artifacts/{appId}/Calibration/{recordId}/{fileId}_{safeFileName}
artifacts/{appId}/Qualification/{recordId}/{fileId}_{safeFileName}
```

Files are not exposed with public download URLs. The application uses Firebase Storage `getBlob()` after authorization, so the Storage Security Rules remain part of the access decision.

## Roles
1. Administrator — user management, workflow administration, audit visibility.
2. Creator — create/edit own studies, upload documents, submit, revise and resubmit.
3. Section Head — review, comment, return, progress to next stage.
4. Deputy QA Manager — review, comment, return, progress to QA Manager.
5. QA Manager — final review, return, approve or reject.
6. Reviewer — Other Department — cross-department review/comment/return/progression.
7. Viewer — read-only access to approved/completed records and permitted final documents.

## Workflow
```text
Creator — Draft
   |
   v
Section Head Review
   |-- Return for Revision --> Creator --> Resubmit
   |
   +-- Other Department Review (when required)
   |
   v
Deputy QA Manager Review
   |-- Return for Revision --> Creator --> Resubmit
   |
   v
QA Manager Review
   |-- Return for Revision --> Creator --> Resubmit
   |-- Reject
   v
Approved / Completed
   |
   v
Viewer read-only access
```

## Review history
Each review/return/approval/rejection creates a separate document in `reviews`. Historical comments are never overwritten by a later comment.

## Audit trail
`audit` is create-only. Ordinary clients cannot update or delete audit documents. The audit document contains UID, display name, role, action, study/module ID, previous/new status, details and a server-generated `createdAt` timestamp.

### GMP/data-integrity note
Firebase client-side Security Rules provide strong access control and append-only behavior, but a production regulated environment should additionally use a trusted server-side audit service (for example Cloud Functions/Admin SDK) to generate authoritative audit events and to prevent a compromised client from submitting misleading event details. The current build improves the rules significantly but should not be treated as a complete 21 CFR Part 11 validation package by itself.

## Deployment
### 1. Replace application file
Replace the existing root `index.html` with the supplied enhanced `index.html`.

### 2. Firestore Rules
In Firebase Console → Firestore Database → Rules, replace the existing rules with `firestore.rules` after reviewing them against your actual project path.

### 3. Storage Rules
In Firebase Console → Storage → Rules, replace the rules with `storage.rules`.

### 4. Authentication
Enable:
- Authentication → Sign-in method → Email/Password.

Do not store passwords in Firestore.

### 5. User profiles
For every existing Firebase Authentication account, create a `users/{uid}` document with a valid role. The old Firestore user documents may remain for historical compatibility; they do not need to be deleted.

### 6. Create the first Administrator
If no UID profile exists yet, create one for the intended administrator directly in Firestore after creating the Firebase Authentication account. The document ID must be the Firebase Authentication UID.

### 7. Storage
No public read access should be enabled. The application expects Firebase Storage Security Rules to control access.

### 8. Hosting deployment
For Firebase Hosting:
```bash
firebase login
firebase use validapp-db
firebase deploy --only hosting,firestore:rules,storage
```
If your project uses a different Hosting target, deploy the Hosting target configured in `firebase.json`.

## Role test plan
### Administrator
- Login with Firebase Authentication.
- Create/edit/disable user profiles.
- Verify user actions appear in audit.
- Verify audit records cannot be edited/deleted.

### Creator
- Create Draft study.
- Upload PDF/DOC/DOCX/XLS/XLSX/JPG/JPEG/PNG.
- Edit Draft.
- Submit to Section Head.
- Receive Returned for Revision.
- Correct and resubmit.
- Verify Creator cannot approve.

### Section Head
- Review assigned pending studies.
- Add review comment.
- Return with mandatory reason.
- Complete review and progress.
- Verify cannot final-approve.

### Reviewer — Other Department
- Review pending cross-department studies.
- Comment/return/progress.
- Verify cannot final-approve.

### Deputy QA Manager
- Review/comment/return/progress.
- Verify cannot final-approve.

### QA Manager
- Review/comment.
- Return with mandatory reason.
- Approve.
- Reject with mandatory reason.
- Verify approved study is protected.

### Viewer
- View approved/completed studies only.
- View/download permitted final documents.
- Verify create/edit/upload/review/approval/user actions are rejected.

## Security tests
Do not rely on hidden buttons. Test direct Firestore/Storage requests using a non-authorized authenticated account. The request must be rejected by Firebase Security Rules.

## Production hardening recommendations
- Enable Firebase App Check for the web application.
- Use Cloud Functions/Admin SDK for authoritative audit event generation.
- Consider restricting Storage file access by exact study workflow state rather than broad role access if your SOP requires assignment-level access.
- Establish a formal validation/CSV package for the application before regulated production use.
- Define retention, backup, disaster recovery, electronic signature and record-retention requirements separately.
