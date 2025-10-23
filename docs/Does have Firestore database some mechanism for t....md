Firestore **does not have a built-in, automatic mechanism** that handles a detailed audit log according to all the requirements you listed.  
However, you can **implement a robust audit logging system** that meets all those requirements by using **Cloud Functions for Firebase** along with Firestore.  
---

## **Implementing Audit Logging with Cloud Functions**

You would primarily use **Cloud Firestore Triggers** within Cloud Functions to monitor changes to your Firestore data and then write the corresponding audit log entries to a separate log collection.

### **1\. The Mechanism**

| Operation | Cloud Function Trigger | Action in the Function |
| :---- | :---- | :---- |
| **CREATE** | onCreate | Write the new document's data to the audit log. The **author** can often be determined from the **Firestore Security Rules** context or from data passed within the write operation itself (e.g., a userId field). |
| **UPDATE** | onUpdate | Write the **new** and **old** document data to the audit log. The change object provided to the function contains both before (old data) and after (new data) snapshots. |
| **DELETE** | onDelete | Write the **deleted document's full data** to the audit log. |

### **2\. Meeting Your Specific Requirements**

| Requirement | How to Achieve it with Cloud Functions |
| :---- | :---- |
| **Detailed log of operations** | The Cloud Function writes a new document to an **audit\_logs** collection for every triggered change. |
| **DateTime and Author** | The function captures the **current server time** (Date.now() or new Date()) for the log entry's timestamp. The **Author** (User ID) needs to be passed *with* the original write operation (e.g., in a lastModifiedBy field on the document) or derived from the **Firebase Authentication** context if using callable functions or security rules to infer the user. |
| **CREATE/UPDATE/DELETE tracking** | Handled directly by the respective onCreate, onUpdate, and onDelete triggers. |
| **UPDATE: Old value** | The onUpdate function trigger provides the change.before snapshot, which is the **old value** of the document before the update. |
| **DELETE: Whole deleted object** | The onDelete function trigger provides the snapshot of the document *just before* it was deleted. You store this entire object in the log. |
| **Objects in the log must be copies (no references)** | By default, when you write the document data (snapshot.data()) from the trigger to the audit\_logs collection, you are writing a **copy** (a new, separate document) of the data at that moment, not a reference. |

In summary, while Firestore lacks the **automatic** feature, **Cloud Functions for Firebase provides the necessary server-side logic and triggers** to implement a comprehensive, customized, and automatic audit logging system that satisfies all your criteria.