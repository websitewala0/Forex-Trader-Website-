````
# Trading Creator — Complete Firebase Backend Architecture, Security Rules & Setup Documentation

## 1. Project Audit & Backend Architecture Overview (Phases 1–3)

`Trading Creator` (`trading_creator.html`) is a high-performance, single-page production web application (`HTML5` + `Tailwind CSS` + `Three.js` 3D `<Lanyard />` + 60fps `Support & Resistance` Physics + `Chart.js` + `Firebase v10 SDK`).

### Connected Firebase Services
1. **Firebase Authentication (`firebase-auth-compat.js`)**:
   - Email & Password registration (`createUserWithEmailAndPassword`) + automatic `sendEmailVerification()`.
   - Email & Password login (`signInWithEmailAndPassword`) + persistent browser sessions.
   - Password reset flow (`sendPasswordResetEmail`) built directly into the Trader Account Portal modal (`#user-reset-pass-btn`).
   - Google OAuth sign-in (`signInWithPopup(new firebase.auth.GoogleAuthProvider())`).
   - Role-based authorization:
     - **Visitor**: Public read-only access to published website content (`publicSettings`, `brokers`, `courses`, `books`, `videos`, `pineScripts`, `pnlEntries`, `quizzes`, `faqs`, `communityMessages`).
     - **Authenticated User (`role: 'user'`)**: Personal isolated Trading Journal (`users/{uid}/tradingJournals`), Quiz attempt history (`quizAttempts`), VIP Broker UID submission (`vipRequests`), and authenticated Community chat/image sharing (`communityMessages`, `communityReports`).
     - **Admin (`request.auth.token.role == 'admin'` or `/admins/{uid}`)**: Full CMS write access across all public collections, VIP Broker UID approval/rejection queue, Quiz/FAQ publisher, and Community moderation.
2. **Cloud Firestore (`firebase-firestore-compat.js`)**:
   - Real-time `onSnapshot` listeners for `publicSettings/siteState` and bounded `communityMessages` (`orderBy('createdAt', 'asc').limitToLast(50)`) with automatic listener cleanup (`tcFirebase.clearListeners()`).
   - Batch seeding (`tcFirebase.pushCollectionsSeed()`) from the Admin Panel (`Section 11: Firebase Backend Configuration & Cloud Sync Studio`).
3. **Firebase Cloud Storage (`firebase-storage-compat.js`)**:
   - Client-side MIME type validation (`image/png`, `image/jpeg`, `image/webp`, `image/gif`, `image/svg+xml`) and 5 MB max file size enforcement (`MAX_UPLOAD_BYTES = 5 * 1024 * 1024`), followed by WebP compression (`640px` max dimension) and automatic upload to Firebase Storage (`public/uploads/...`, `community/{uid}/...`, `journals/{uid}/...`).

---

## 2. Cloud Firestore Data Model & Collection Schema (Phase 9)

| Collection / Path | Purpose | Key Fields & Types | Ownership | Read Access | Write Access | Indexes Required |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `publicSettings/siteState` | Master public site configuration & state | `lanyardPhoto: string`, `heroLogos: array`, `flipBrokers: array`, `socialBoxes: array`, `courses: array`, `books: array`, `videos: array`, `pineScripts: array`, `pnlItems: array`, `quizzes: array`, `faqs: array` | Admin | Public (`true`) | Admin Only (`isAdmin()`) | Single-doc lookup |
| `admins/{adminUid}` | Server-validated admin UID registry | `email: string`, `grantedAt: timestamp` | System / Admin SDK | Owner or Admin | `false` (Admin SDK / Cloud Functions only) | Single-doc lookup |
| `users/{userId}` | Authenticated trader profile & synced journal | `uid: string`, `name: string`, `email: string`, `role: 'user'`, `vipApproved: bool`, `journalTrades: array`, `updatedAt: number` | `userId` | `isOwner(userId) \|\| isAdmin()` | Create/Update by Owner (cannot modify `role`, `vipApproved`, `paymentStatus`) | Single-doc lookup |
| `users/{userId}/tradingJournals/{tradeId}` | Private per-user trade execution records | `date: string`, `market: string`, `symbol: string`, `side: 'LONG'\|'SHORT'`, `qty: number`, `entry: number`, `exit: number`, `sl: number`, `setup: string`, `mindset: string`, `notes: string` | `userId` | `isOwner(userId)` (Strictly Private) | `isOwner(userId)` | `date DESC` |
| `brokers/{brokerId}` | Partner brokers for Hero, Flip & Folder | `name: string`, `url: string`, `logo: string`, `tagline: string`, `order: number`, `enabled: bool` | Admin | Public (`true`) | Admin Only (`isAdmin()`) | `enabled ASC, order ASC` |
| `courses/{courseId}` | Mentorship & price-action courses | `title: string`, `cat: string`, `badge: string`, `price: string`, `duration: string`, `features: array`, `published: bool`, `order: number` | Admin | `resource.data.published == true \|\| isAdmin()` | Admin Only (`isAdmin()`) | `published ASC, cat ASC, order ASC` |
| `books/{bookId}` | 3D Bookshelf items | `title: string`, `subtitle: string`, `price: string`, `pages: string`, `url: string`, `summary: string`, `cover: string`, `published: bool` | Admin | Public (`true`) | Admin Only (`isAdmin()`) | `order ASC` |
| `videos/{videoId}` | Learning YouTube video embeds | `embedUrl: string`, `rawEmbed: string`, `order: number` | Admin | Public (`true`) | Admin Only (`isAdmin()`) | `order ASC` |
| `pineScripts/{scriptId}` | Copyable TradingView Pine Script v5 cards | `title: string`, `tag: string`, `img: string`, `code: string` | Admin | Public (`true`) | Admin Only (`isAdmin()`) | `order ASC` |
| `pnlEntries/{entryId}` | Circular Carousel P&L screenshots | `title: string`, `subtitle: string`, `img: string`, `customUploaded: bool`, `published: bool`, `order: number` | Admin | Public (`true`) | Admin Only (`isAdmin()`) | `published ASC, order ASC` |
| `quizzes/{quizId}` | Interactive Trading Quiz questions | `q: string`, `options: array`, `correct: number`, `exp: string`, `published: bool` | Admin | Public (`true`) | Admin Only (`isAdmin()`) | `order ASC` |
| `quizAttempts/{attemptId}` | User quiz score history | `uid: string`, `score: number`, `total: number`, `completedAt: number` | `uid` | `resource.data.uid == request.auth.uid \|\| isAdmin()` | Create by Owner (`request.resource.data.uid == request.auth.uid`) | `uid ASC, completedAt DESC` |
| `vipRequests/{reqId}` | Partner Broker UID verification queue | `id: string`, `userUid: string`, `broker: string`, `brokerUid: string`, `status: 'pending'\|'approved'\|'rejected'`, `submittedAt: string` | `userUid` | Owner or Admin | Create by Owner (`status == 'pending'`); Update/Approve by Admin Only | `status ASC, submittedAt DESC` |
| `communityMessages/{msgId}` | Real-time trader chat messages | `docId: string`, `uid: string`, `channel: string`, `user: string`, `role: string`, `text: string`, `img: string`, `reactions: map`, `createdAt: number` | `uid` | Public (`true`) | Create by Signed-In User (`text.size() <= 500`); Delete by Owner or Admin | `channel ASC, createdAt ASC` |
| `communityReports/{repId}` | Flagged community messages for moderation | `id: string`, `channel: string`, `author: string`, `text: string`, `reportedBy: string`, `createdAt: string` | `reportedBy` | Admin Only (`isAdmin()`) | Create by Signed-In User; Update/Delete by Admin | `createdAt DESC` |
| `faqs/{faqId}` | Frequently Asked Questions | `q: string`, `a: string`, `order: number` | Admin | Public (`true`) | Admin Only (`isAdmin()`) | `order ASC` |

---

## 3. Production Security Rules (`firestore.rules` & `storage.rules`)

### 3.1 `firestore.rules` (Default-Deny, Role & Ownership Enforced)

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isSignedIn() {
      return request.auth != null;
    }
    function isOwner(userId) {
      return isSignedIn() && request.auth.uid == userId;
    }
    function isAdmin() {
      return isSignedIn() && (
        request.auth.token.role == 'admin' ||
        exists(/databases/$(database)/documents/admins/$(request.auth.uid))
      );
    }
    function validString(val, minLen, maxLen) {
      return val is string && val.size() >= minLen && val.size() <= maxLen;
    }

    // Default-deny all unmatched documents
    match /{document=**} {
      allow read, write: if false;
    }

    // Admin UID registry (Only writable via Firebase Admin SDK / Cloud Functions)
    match /admins/{adminUid} {
      allow read: if isOwner(adminUid) || isAdmin();
      allow write: if false;
    }

    // Public Website Content Collections (Public Read, Admin-Only Write)
    match /publicSettings/{docId} {
      allow read: if true;
      allow write: if isAdmin();
    }
    match /brokers/{brokerId} {
      allow read: if true;
      allow write: if isAdmin();
    }
    match /courses/{courseId} {
      allow read: if resource.data.published == true || isAdmin();
      allow write: if isAdmin();
    }
    match /memberships/{tierId} {
      allow read: if true;
      allow write: if isAdmin();
    }
    match /books/{bookId} {
      allow read: if true;
      allow write: if isAdmin();
    }
    match /videos/{videoId} {
      allow read: if true;
      allow write: if isAdmin();
    }
    match /pineScripts/{scriptId} {
      allow read: if true;
      allow write: if isAdmin();
    }
    match /pnlEntries/{entryId} {
      allow read: if true;
      allow write: if isAdmin();
    }
    match /quizzes/{quizId} {
      allow read: if true;
      allow write: if isAdmin();
    }
    match /faqs/{faqId} {
      allow read: if true;
      allow write: if isAdmin();
    }

    // User Profiles & Strictly Isolated Private Trading Journal
    match /users/{userId} {
      allow read: if isOwner(userId) || isAdmin();
      allow create: if isOwner(userId)
        && (!('role' in request.resource.data) || request.resource.data.role == 'user')
        && (!('vipApproved' in request.resource.data) || request.resource.data.vipApproved == false);
      allow update: if (isOwner(userId)
        && !request.resource.data.diff(resource.data).affectedKeys().hasAny(['role', 'vipApproved', 'paymentStatus']))
        || isAdmin();
      allow delete: if isAdmin();

      match /tradingJournals/{tradeId} {
        allow read, write: if isOwner(userId);
      }
      match /tradingPerformance/{recordId} {
        allow read, write: if isOwner(userId);
      }
    }

    // Quiz Attempts
    match /quizAttempts/{attemptId} {
      allow read: if isSignedIn() && (resource.data.uid == request.auth.uid || isAdmin());
      allow create: if isSignedIn() && request.resource.data.uid == request.auth.uid;
      allow update, delete: if isAdmin();
    }

    // VIP Broker UID Verification Requests
    match /vipRequests/{reqId} {
      allow read: if isSignedIn() && (resource.data.userUid == request.auth.uid || isAdmin());
      allow create: if isSignedIn()
        && request.resource.data.userUid == request.auth.uid
        && request.resource.data.status == 'pending'
        && validString(request.resource.data.brokerUid, 4, 32);
      allow update, delete: if isAdmin();
    }

    // Community Chat & Moderation Reports
    match /communityMessages/{msgId} {
      allow read: if true;
      allow create: if isSignedIn()
        && request.resource.data.uid == request.auth.uid
        && validString(request.resource.data.text, 1, 500);
      allow update, delete: if isSignedIn() && (resource.data.uid == request.auth.uid || isAdmin());
    }
    match /communityReports/{repId} {
      allow read, update, delete: if isAdmin();
      allow create: if isSignedIn() && request.resource.data.reportedBy == request.auth.uid;
    }
  }
}


````

### 3.2 `storage.rules` (MIME Type, 5MB Size & Ownership Enforced)

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    function isSignedIn() {
      return request.auth != null;
    }
    function isAdmin() {
      return isSignedIn() && request.auth.token.role == 'admin';
    }
    function isValidImage() {
      return request.resource.size <= 5 * 1024 * 1024
        && request.resource.contentType.matches('image/(png|jpeg|jpg|webp|gif|svg\\+xml)');
    }

    match /public/{allPaths=**} {
      allow read: if true;
      allow write: if isAdmin() && isValidImage();
    }

    match /community/{userId}/{fileName} {
      allow read: if isSignedIn();
      allow write: if isSignedIn() && request.auth.uid == userId && isValidImage();
      allow delete: if (isSignedIn() && request.auth.uid == userId) || isAdmin();
    }

    match /journals/{userId}/{fileName} {
      allow read, write: if isSignedIn() && request.auth.uid == userId && isValidImage();
    }

    match /{allPaths=**} {
      allow read, write: if false;
    }
  }
}


```

## 4. Safe First-Admin Bootstrap & Cloud Functions (`functions/index.js`)

Never assign admin privileges from client-side code. Use the Firebase Admin SDK script or Cloud Function below to bootstrap your first Admin user:

```
// scripts/bootstrap-admin.js (Run once with Node.js using your Service Account key on your workstation)
const admin = require('firebase-admin');
admin.initializeApp({ credential: admin.credential.applicationDefault() });

async function grantAdminRole(email) {
  const user = await admin.auth().getUserByEmail(email);
  await admin.auth().setCustomUserClaims(user.uid, { role: 'admin' });
  await admin.firestore().collection('admins').doc(user.uid).set({
    email: user.email,
    grantedAt: admin.firestore.FieldValue.serverTimestamp()
  });
  console.log(`Granted custom claim role="admin" to ${email} (${user.uid})`);
}

grantAdminRole('signupallplatform@gmail.com').catch(console.error);


```

## 5. Step-by-Step Firebase Console Setup Instructions

1. **Create a Firebase Project**:

   * Open [Firebase Console](https://console.firebase.google.com/), create a new project (e.g., `trading-creator-prod`), and register a **Web App (`</>`)**.

2. **Enable Authentication Providers**:

   * In **Build → Authentication → Sign-in method**, enable **Email/Password** and **Google**.

3. **Create Firestore Database & Storage Bucket**:

   * In **Build → Firestore Database**, click **Create database** in production mode and paste the `firestore.rules` above into the **Rules** tab (or click **`Copy Firestore & Storage Security Rules`** in Section 11 of the Admin Panel).

   * In **Build → Storage**, click **Get started** and paste the `storage.rules` above into the **Rules** tab.

4. **Connect Your Website in 1 Click**:

   * Sign in to the Admin Panel on your website, scroll to **Section 11: Firebase Backend Configuration & Cloud Sync Studio**, paste your `firebaseConfig` JSON, click **Connect & Initialize Firebase**, and click **Push Current Website Data to Firestore**.

```


### Admin Forgot Email / Forgot Password & Google Account OAuth (User + Admin)
- **Admin Forgot Password (`#admin-forgot-pass-btn`)**:
  - Supports **Firebase Password Reset Email** (`tcFirebase.auth.sendPasswordResetEmail`) when Firebase is connected.
  - Supports **Direct Admin Password Reset** after verifying the registered Admin Email (`#admin-rec-email-verify` + `#admin-rec-new-pass`).
- **Admin Forgot Email (`#admin-forgot-email-btn`)**:
  - Displays a privacy-safe masked email hint (`si********m@gmail.com`) so the full Admin Email is never exposed publicly.
  - Verifies the current Admin Password (`#admin-rec-pass-verify`) to autofill the registered Admin Email or update to a new Admin Email (`#admin-rec-new-email`).
- **Google Account Authentication (`#user-google-signin-btn` & `#admin-google-signin-btn`)**:
  - Both the **User Sign-In Modal** and **Admin Sign-In Modal** feature official **Continue with Google** OAuth buttons.
  - When Firebase is connected, uses `firebase.auth().signInWithPopup(new firebase.auth.GoogleAuthProvider())` (with Admin verifying `claims.role === "admin"`, `/admins/{uid}`, or matching the authorized Admin Google email).
  - Includes an interactive **Google Account OAuth Chooser (`#google-oauth-modal`)** fallback for local/preview mode or iframe environments where third-party popups are restricted.

---

## 6. Production Google Authentication (`GoogleAuthProvider`) & Testing Guide

### 6.1 Implementation Architecture (`trading_creator.html`)
1. **Official Firebase SDK Integration**:
   - Uses `firebase-app-compat.js` and `firebase-auth-compat.js` (`v10.12.2`) without creating duplicate Firebase apps (`window.firebase.apps.length ? window.firebase.app() : window.firebase.initializeApp(configObj)`).
2. **Desktop `signInWithPopup` & Mobile `signInWithRedirect`**:
   - Configures `new window.firebase.auth.GoogleAuthProvider()` with `profile` and `email` scopes and `prompt: "select_account"`.
   - On desktop browsers, executes `tcFirebase.auth.signInWithPopup(provider)` (with automatic fallback to `signInWithRedirect(provider)` if the browser blocks popups via `auth/popup-blocked`).
   - On standalone mobile viewports (`isMobileViewportOrUA()`), uses `tcFirebase.auth.signInWithRedirect(provider)` and processes the returning OAuth result via `tcFirebase.auth.getRedirectResult()` on startup.
   - **Zero Simulated Logins**: If Firebase is not configured yet, clicking **Sign in with Google** or **Sign up with Google** does not fake a login; it reports the required Firebase configuration (`apiKey`, `authDomain`, `projectId`) and opens the configuration modal (`#google-oauth-modal`).
3. **Session Persistence, Profile Picture (`photoURL`), Name & Email Display**:
   - Enforces `firebase.auth.Auth.Persistence.LOCAL` and listens to `tcFirebase.auth.onAuthStateChanged` to maintain sessions across page refreshes.
   - Displays the authenticated user's Google `photoURL` (`referrerpolicy="no-referrer"` with initial-avatar fallback), `displayName`, and `email` in:
     - Top Navigation Bar (`#nav-user-profile-pill`)
     - Navigation Menu Drawer (`#drawer-signed-user-card`)
     - Trader Account Modal (`#user-authenticated-profile-card`)
     - Protected Trading Journal Header (`#journal-auth-profile-bar`)
4. **Protected Private Routes & Automatic Dashboard Redirects**:
   - Navigating to `#journal` (*Personal Trading Journal*) or clicking *My Personal Stats (Synced)* while unauthenticated intercepts the route, stores `pendingProtectedRoute = "journal"`, and opens the Sign-In modal.
   - Navigating to `#admin` (*Admin CMS Studio*) while unauthenticated opens the Admin Sign-In modal.
   - After successful authentication, users are automatically redirected to their requested protected route (`#journal` for Traders or `#admin` for verified Admins).
5. **Safe Non-Destructive Firestore Profile Upsert (`upsertUserProfileSafely`)**:
   - Keyed by the authenticated Firebase `user.uid` (`users/{uid}`).
   - Checks `users/{uid}` first and merges only profile metadata (`uid`, `name`, `email`, `photoURL`, `providerId`, `updatedAt`, `lastLoginAt`) so existing `journalTrades`, `role`, and `vipApproved` fields are never overwritten.
   - Never stores passwords, OAuth `accessToken`, or `idToken` in Firestore or `localStorage`.


```