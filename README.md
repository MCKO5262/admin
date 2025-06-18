
# Firebase Studio

This is a NextJS starter in Firebase Studio.

To get started, take a look at src/app/page.tsx.

## Proposed Firestore Database Structure

This section outlines the recommended Firestore database structure for the application.

### `admins` Collection (Formerly `users`)

Stores admin user profile information and roles for the CMS. This collection is distinct from any general user collection your main application might have.

*   **Document ID:** `uid` (from Firebase Authentication)
*   **Fields:**
    *   `email: string` (Admin's email)
    *   `name: string` (Admin's display name)
    *   `role: string` (e.g., "Super Admin", "Sub Admin" - matching `UserRole` enum from `src/types/index.ts`)
    *   `avatar: string` (URL to avatar image, optional)
    *   `createdAt: firebase.firestore.Timestamp` (Server timestamp when the admin document was created)
    *   `updatedAt: firebase.firestore.Timestamp` (Server timestamp when the admin document was last updated, optional)

*This collection is utilized by `src/contexts/auth-context.tsx` for managing admin user sessions and roles within the admin panel.*

### `categories` Collection

Stores definitions for content categories (e.g., Blog Posts, Products).

*   **Document ID:** Auto-generated Firestore ID (recommended)
*   **Fields:**
    *   `name: string` (e.g., "Blog Posts")
    *   `slug: string` (URL-friendly identifier, e.g., "blog-posts". Should be unique if used for routing.)
    *   `description: string` (Optional, a brief description of the category)
    *   `fields: array` (An array of field definition objects, mirroring `FieldDefinition` from `src/types/index.ts`)
        *   Each object in the array: `{ id: string, label: string, type: string (matching FieldType enum), required: boolean, placeholder: string (optional) }`
    *   `createdAt: firebase.firestore.Timestamp`
    *   `updatedAt: firebase.firestore.Timestamp`
    *   `entryCount: number` (Optional: for quickly displaying how many entries belong to this category. Can be updated with Cloud Functions or batched writes for consistency.)


### `entries` Collection

Stores individual content entries belonging to a specific category.

*   **Document ID:** Auto-generated Firestore ID
*   **Fields:**
    *   `categoryId: string` (ID of the document in the `categories` collection this entry belongs to)
    *   `title: string` (A general title for the entry, useful for display in lists and admin UI. This might be derived from a specific field in `data` if a standard 'title' field exists in the category definition, or can be a separate field.)
    *   `data: map` (An object where keys are `fieldDefinition.id` from the parent category's `fields` array, and values are the actual content for those fields.)
        *   Example: `data: { title: "My First Blog Post", content: "...", authorName: "John Doe" }`
    *   `status: string` (e.g., "draft", "published", "scheduled" - matching `Entry['status']` type)
    *   `publishAt: firebase.firestore.Timestamp` (Optional, used if `status` is "scheduled")
    *   `createdAt: firebase.firestore.Timestamp`
    *   `updatedAt: firebase.firestore.Timestamp`
    *   `createdBy: string` (UID of the admin user who created the entry, optional)
    *   `slug: string` (Optional: URL-friendly identifier for the entry, if needed for public-facing URLs. Could be derived from a title field.)

### `users` Collection (For App Users, not Admins)

Stores profile information for the end-users of your application, primarily for push notification targeting.

*   **Document ID:** `uid` (from Firebase Authentication of the app user)
*   **Fields:**
    *   `email: string` (App user's email)
    *   `displayName: string` (App user's display name, optional)
    *   `fcmTokens: array<string>` (List of Firebase Cloud Messaging registration tokens for this user's devices. Updated by the client app.)
    *   `createdAt: firebase.firestore.Timestamp` (Server timestamp when the app user document was created)
    *   `lastLoginAt: firebase.firestore.Timestamp` (Optional: Server timestamp of the user's last login)

### `notifications` Collection (For Push Notification Logs)

Stores logs of push notifications sent or scheduled to be sent to app users. A Firebase Function will typically process these.

*   **Document ID:** Auto-generated Firestore ID
*   **Fields:**
    *   `title: string` (Notification title)
    *   `body: string` (Notification body/message)
    *   `imageUrl: string` (Optional URL for an image in the notification)
    *   `deepLink: string` (Optional deep link URL for the app)
    *   `scheduleAt: firebase.firestore.Timestamp` (Optional: If set, the Firebase Function should send at this time)
    *   `adminCreator: map`
        *   `uid: string` (UID of the admin who created the notification request)
        *   `email: string` (Email of the admin)
        *   `name: string` (Name of the admin, optional)
    *   `createdAt: firebase.firestore.Timestamp` (Timestamp of when the admin created this log)
    *   `processingStatus: string` (e.g., "pending", "processing", "completed", "partially_completed", "error" - managed by the Firebase Function)
    *   `processedAt: firebase.firestore.Timestamp` (Optional: Timestamp of when processing by Firebase Function started/completed)
    *   `targets: array` (Array of target user tokens and their individual statuses)
        *   Each object in the array:
            *   `userId: string` (Target app user's UID)
            *   `userEmail: string` (Target app user's email, denormalized)
            *   `userName: string` (Target app user's name, denormalized, optional)
            *   `token: string` (The specific FCM token targeted for this user device)
            *   `status: string` (e.g., "pending", "success", "failed" - updated by the Firebase Function)
            *   `error: string` (Optional: Error message if sending to this token failed)
            *   `messageId: string` (Optional: FCM message ID on successful send)
            *   `attemptedAt: firebase.firestore.Timestamp` (Optional: When the Firebase Function attempted to send to this token)

This structure is designed to be scalable and flexible, allowing for dynamic content types based on category definitions. Firestore security rules should be configured to protect this data appropriately (e.g., only authenticated admins can write to `admins`, `categories`, `entries`).

## Firestore Indexes

As you build queries for Firestore, you might encounter errors like **"FirebaseError: The query requires an index."** This means you need to create a composite index in your Firebase console for that specific query to work efficiently. The error message in your Next.js development server console will usually provide a direct link to create the missing index.

---

🚨🚨🚨 **CRITICAL: FIXING THE RECURRING "The query requires an index" ERROR FOR `entries` COLLECTION** 🚨🚨🚨

If you are seeing an error message in your Next.js console similar to this (especially when navigating to the "Entries" page or filtering entries):

```
Console Error: FirebaseError: The query requires an index. You can create it here: https://console.firebase.google.com/v1/r/project/YOUR_PROJECT_ID/firestore/indexes?create_composite=...
```
Or this, coming from the application's own error handling in `src/lib/actions/entryActions.ts`:
```
Console Error: Firestore query requires an index. Please check Firebase console for index creation link specific to this query (likely on 'entries' collection for 'categoryId' and 'createdAt').
```

This **SPECIFICALLY MEANS** the query on the `entries` collection (likely in `src/lib/actions/entryActions.ts` -> `getEntries` when filtering by `categoryId` and ordering by `createdAt`) **NEEDS AN INDEX CONFIGURED IN YOUR FIREBASE PROJECT.**

**THIS IS NOT A CODE BUG IN THE NEXT.JS APPLICATION THAT CAN BE FIXED WITH CODE CHANGES. YOU MUST PERFORM THIS ACTION IN THE FIREBASE CONSOLE.**

**To fix this (for the `setgelzuin-app` project, as indicated by YOUR error message):**

1.  **CLICK THE LINK PROVIDED IN YOUR ERROR MESSAGE IN THE CONSOLE.** For the `setgelzuin-app` project, the specific link you are encountering is:
    `https://console.firebase.google.com/v1/r/project/setgelzuin-app/firestore/indexes?create_composite=Ck5wcm9qZWN0cy9zZXRnZWx6dWluLWFwcC9kYXRhYmFzZXMvKGRlZmF1bHQpL2NvbGxlY3Rpb25Hcm91cHMvZW50cmllcy9pbmRleGVzL18QARoOCgpjYXRlZ29yeUlkEAEaDQoJY3JlYXRlZEF0EAIaDAoIX19uYW1lX18QAg`
2.  This link will take you directly to the Firebase Console page to create the required composite index for the `entries` collection.
3.  The fields for the index will be pre-filled based on the link:
    *   **Collection:** `entries` (This is a collection group query, so it applies to all collections named 'entries')
    *   **Fields to index:**
        *   `categoryId` (Ascending / Өсөхөөр)
        *   `createdAt` (Descending / Буурахаар)
4.  Click **"Create Index"** (or the equivalent in your language).
5.  **VERY IMPORTANT: WAIT for the index to build.** This might take **SEVERAL MINUTES (sometimes 5-10 minutes or more depending on data size).** You can see the status in the Firebase Console (it will go from "Building" to "Ready" or "Enabled"). **The error will persist until the index is fully built and enabled.** Refreshing the app too early will not solve the issue.
6.  Once the index status is **"Ready" / "Enabled"** in the Firebase Console, **THEN** refresh your application. The error should be gone.

**Always check the Firebase console or your server logs for specific index creation links if you encounter these errors. This is a Firebase Firestore configuration step, not a code change within the Next.js application itself.**

---

## Firestore Security Rules

**Example Firestore Security Rules Snippet (Conceptual):**
```firestore
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Admins collection
    match /admins/{adminId} {
      allow read: if request.auth != null && request.auth.uid == adminId; // Own record
      allow list, write: if request.auth != null && get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Super Admin';
    }

    // Categories collection
    match /categories/{categoryId} {
      allow read: if request.auth != null; // Authenticated admins can read
      allow list, create, update, delete: if request.auth != null && 
                                          (get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Super Admin' ||
                                           get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Sub Admin');
    }

    // Entries collection
    match /entries/{entryId} {
      allow read: if request.auth != null; // Authenticated admins can read
      allow list, create, update, delete: if request.auth != null &&
                                          (get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Super Admin' ||
                                           get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Sub Admin');
    }

    // App Users collection
    match /users/{userId} {
      // Admins can read app user data for notification targeting
      allow read: if request.auth != null && 
                    (get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Super Admin' ||
                     get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Sub Admin');
      // Only the app user themselves can write to their own document (e.g., update fcmTokens)
      // This rule assumes app users authenticate with Firebase Auth as well.
      allow write: if request.auth != null && request.auth.uid == userId;
      // Listing app users should be restricted to admins.
      allow list: if request.auth != null && 
                    (get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Super Admin' ||
                     get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Sub Admin');
    }

    // Notifications collection
    match /notifications/{notificationId} {
      // Admins can create notification requests
      allow create: if request.auth != null &&
                      (get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Super Admin' ||
                       get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Sub Admin');
      // Admins can read notification logs
      allow read, list: if request.auth != null &&
                          (get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Super Admin' ||
                           get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Sub Admin');
      // Only backend (e.g., Firebase Function with elevated privileges) should update status.
      // This is a simplified rule; a more robust one might check for a specific service account UID.
      allow update: if request.auth == null || get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Super Admin'; // Or check for service account
      allow delete: if request.auth != null && get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'Super Admin';
    }
  }
}
```
*Remember to tailor security rules to your specific application needs.*

## Environment Variables

Ensure you have a `.env.local` file in the root of your project with your Firebase project configuration:

```env
NEXT_PUBLIC_FIREBASE_API_KEY=YOUR_API_KEY
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=YOUR_AUTH_DOMAIN
NEXT_PUBLIC_FIREBASE_PROJECT_ID=YOUR_PROJECT_ID
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=YOUR_STORAGE_BUCKET
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=YOUR_MESSAGING_SENDER_ID
NEXT_PUBLIC_FIREBASE_APP_ID=YOUR_APP_ID
NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID=YOUR_MEASUREMENT_ID (optional)
```
Replace `YOUR_...` with your actual Firebase project credentials.
Remember to restart your development server (`npm run dev`) after creating or modifying `.env.local`.

```