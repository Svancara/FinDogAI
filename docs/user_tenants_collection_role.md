- The following text describes the first draft of the FinDogAI application.
- The first draft used a term "customer". Our new version uses the term "user".
- The first draft used an OAuth login mechanism (Google, Facebook, Twitter, etc.) which returned a token which was used to authenticate the user.
- The token was mapped to a Firebase user ID which was used to access the Firestore database.
- I guess the reason was that OAuth login token is not suitable to be used as a primary key in the Firestore database.
The primary key in the Firestore database was in uuid format.

Please read this text and determine if we will need to use a similar mechanism for the new version of the FinDogAI application, which will use Firebase Authentication together with Google Sign In and Apple Sign In in the future too.

If so, please modify PRD.MD to describe the similar approach using the `user_tenants` collection in the new version of the application.
Write needed epics and stories to implement the changes.

The description of the `user_tenants` collection role follows:

# Role of the `user_tenants` Firestore Collection

The **`user_tenants`** collection serves as a **user-to-tenant mapping table** that bridges Firebase Authentication users with the multi-tenant architecture of FinDogAI.

## Purpose

**Primary Function**: Maps a Firebase user ID to their corresponding tenant ID, enabling multi-tenant isolation while using Firebase Authentication.

## Structure

Each document in the collection:
- **Document ID**: Firebase user ID (UID)
- **Fields**:
  - `tenantId` (string): The UUID of the tenant/customer this user belongs to
  - `createdAt` (DateTime): When the mapping was created

## How It Works

### 1. **User Sign-In Flow**
When a user signs in (via Firebase or dev signup):
1. The system extracts the Firebase user ID from the authentication token
2. Looks up `user_tenants/{userId}` to find their tenant ID
3. If the mapping exists, retrieves the `tenantId`
4. If not, creates a new Customer (which generates a new tenant) and stores the mapping

### 2. **Tenant Provisioning**
The `UserTenantService.GetOrCreateTenantForUserAsync()` method:
- Checks if a mapping exists for the user
- If yes: Returns the existing tenant ID
- If no: Creates a new Customer via CQRS (which becomes the tenant) and saves the mapping

### 3. **JWT Token Generation**
Once the tenant ID is resolved, it's embedded in the JWT token as a custom claim:
```json
{
  "user_id": "firebase-uid",
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "roles": ["user"]
}
```

## Security & Isolation

- **Tenant Isolation**: Every subsequent API request uses the `tenant_id` claim to scope data access
- **Firestore Rules**: Enforce that users can only access data under `customers/{tenantId}` where `tenantId` matches their token claim
- **One User = One Tenant**: Currently, each user maps to exactly one tenant (single-tenant per user model)

## Key Use Cases

1. **First-time user registration**: Creates both the user-tenant mapping and a new Customer/tenant
2. **Returning user login**: Quickly resolves which tenant the user belongs to
3. **Duplicate prevention**: Checks if a user already has a tenant during signup (returns 409 Conflict)

This design allows FinDogAI to use Firebase Authentication for user management while maintaining strict multi-tenant data isolation at the Firestore level.
