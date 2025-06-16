
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

This structure is designed to be scalable and flexible, allowing for dynamic content types based on category definitions. Firestore security rules should be configured to protect this data appropriately (e.g., only authenticated admins can write to `admins`, `categories`, `entries`).

## Firestore Indexes

As you build queries for Firestore, you might encounter errors like **"FirebaseError: The query requires an index."** This means you need to create a composite index in your Firebase console for that specific query to work efficiently. The error message in your Next.js development server console will usually provide a direct link to create the missing index.

### **CRITICAL: Fixing "The query requires an index" Error for `entries` Collection**

If you see an error message in your console similar to this (especially when navigating to the "Entries" page or filtering entries):

```
Console Error: FirebaseError: The query requires an index. You can create it here: https://console.firebase.google.com/v1/r/project/YOUR_PROJECT_ID/firestore/indexes?create_composite=...
```
Or this, coming from the application's own error handling in `src/lib/actions/entryActions.ts`:
```
Console Error: Firestore query requires an index. Please check Firebase console for index creation link specific to this query (likely on 'entries' collection for 'categoryId' and 'createdAt').
```

This **specifically means** the query on the `entries` collection (likely in `src/lib/actions/entryActions.ts` -> `getEntries` when filtering by `categoryId` and ordering by `createdAt`) **needs an index configured in your Firebase project.**

**To fix this (for the `setgelzuin-app` project, as indicated by YOUR error message):**

1.  **Click the link provided in YOUR error message in the console.** For the `setgelzuin-app` project, the specific link you are encountering is:
    `https://console.firebase.google.com/v1/r/project/setgelzuin-app/firestore/indexes?create_composite=Ck5wcm9qZWN0cy9zZXRnZWx6dWluLWFwcC9kYXRhYmFzZXMvKGRlZmF1bHQpL2NvbGxlY3Rpb25Hcm91cHMvZW50cmllcy9pbmRleGVzL18QARoOCgpjYXRlZ29yeUlkEAEaDQoJY3JlYXRlZEF0EAIaDAoIX19uYW1lX18QAg`
2.  This link will take you directly to the Firebase Console page to create the required composite index for the `entries` collection.
3.  The fields for the index will be pre-filled based on the link:
    *   **Collection:** `entries` (This is a collection group query, so it applies to all collections named 'entries')
    *   **Fields to index:**
        *   `categoryId` (Ascending / Өсөхөөр)
        *   `createdAt` (Descending / Буурахаар)
4.  Click **"Create Index"** (or the equivalent in your language).
5.  **WAIT for the index to build.** This might take **several minutes**. You can see the status in the Firebase Console (it will go from "Building" to "Ready" or "Enabled"). **The error will persist until the index is fully built and enabled.** Refreshing the app too early will not solve the issue.
6.  Once the index status is **"Ready" / "Enabled"**, refresh your application. The error should be gone.

**Always check the Firebase console or your server logs for specific index creation links if you encounter these errors. This is a Firebase Firestore configuration step, not a code change within the Next.js application itself.**

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


    
