The most straightforward way to implement an integer counter autoincremented on the server side in **Firebase/Firestore** is by using the **`FieldValue.increment()`** method.

This method atomically updates a numeric field on the server without needing to read the current value first, which prevents race conditions where multiple clients try to increment at the same time and overwrite each other's changes.

Here is an explanation of the implementation and considerations for different traffic levels:

-----

## 1\. Low to Moderate Traffic: `FieldValue.increment()`

For most applications where the counter is updated less frequently than once per second on the same document, using `FieldValue.increment()` is the ideal solution. It is efficient, atomic, and simple to implement from either the client or a server environment (like **Cloud Functions** or a custom backend using the Admin SDK).

### Implementation

You can call this method directly on a document reference:

```javascript
// Get a reference to the document holding the counter
const counterRef = db.collection('metadata').doc('stats');

// Atomically increment the 'count' field by 1
counterRef.update({
    count: firebase.firestore.FieldValue.increment(1)
})
.then(() => {
    console.log("Counter successfully incremented!");
})
.catch((error) => {
    console.error("Error incrementing counter: ", error);
});
```

  * **Server-Side:** When running this code in a secure environment like a **Firebase Cloud Function** using the **Admin SDK**, the increment is guaranteed to be atomic, and you have server-side control over who can trigger the update.
  * **Client-Side:** You can also run this from a client application, but you **must** use **Firestore Security Rules** to ensure that users can *only* apply the increment operation and cannot set the counter to an arbitrary value.

-----

## 2\. High Traffic: Distributed Counters

Firestore has a write limit of about **one update per second per document**. If your application requires a higher write rate (e.g., hundreds of updates per second) to a single counter, you must use a **Distributed Counter**.

A distributed counter splits the total count across multiple documents, called **shards**, to distribute the write load.

### Implementation Concept

1.  **Counter Structure:** Create a main counter document (e.g., `counters/my_count`) with a **subcollection of shards** (e.g., `shards/0`, `shards/1`, `shards/2`, etc.). Each shard document has its own `count` field, initialized to `0`.

2.  **Incrementing:** To increment the total count, select a shard **randomly** and use `FieldValue.increment(1)` on that shard's `count` field.

    ```javascript
    const numShards = 10; // Define a number of shards
    const counterRef = db.collection('counters').doc('my_count');

    // 1. Select a shard randomly
    const shardId = Math.floor(Math.random() * numShards).toString();
    const shardRef = counterRef.collection('shards').doc(shardId);

    // 2. Atomically increment the count on the selected shard
    shardRef.update({
        count: firebase.firestore.FieldValue.increment(1)
    });
    ```

3.  **Reading the Total:** To get the total count, you must **read all the shard documents** in the subcollection and **sum their values** on the client or server.

-----

## 3\. Atomic Read-Modify-Write: Transactions

While **`FieldValue.increment()`** handles simple incrementing, if you need to perform an operation where the increment depends on the document's current state (e.g., "only increment if the count is less than 100"), you must use a **Firestore Transaction**.

A transaction is an atomic batch of read and write operations that ensures that if any document read during the transaction is modified by another process before the transaction commits, the transaction will **automatically retry**.

### Implementation Concept

1.  **Define a Transaction:** Use the `runTransaction()` method.
2.  **Read:** Inside the transaction, read the counter document.
3.  **Compute/Conditional Logic:** Apply any custom logic (e.g., check limits, calculate a new value).
4.  **Write:** Write the new value back to the document.

**Example using the Admin SDK in a Cloud Function:**

```javascript
const counterRef = db.collection('metadata').doc('stats');

await db.runTransaction(async (transaction) => {
    const doc = await transaction.get(counterRef);

    if (!doc.exists) {
        // Handle case where document doesn't exist
        return; 
    }

    const currentCount = doc.data().count || 0;
    
    // Custom logic: Only increment if count is less than 100
    if (currentCount < 100) {
        const newCount = currentCount + 1;
        transaction.update(counterRef, { count: newCount });
    }
});
```

  * **Note:** Transactions should be avoided for high-frequency counters because they also count towards the one-write-per-second limit on the counter document. Use `FieldValue.increment()` unless you have a necessary conditional logic.