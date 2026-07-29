# Offline-First Syncing in Agriculture SaaS: Building a Background Request Queue with Dexie.js & IndexedDB

When building software for the agricultural sector—in our case, **ASFControl**, a SaaS for managing native stingless bees (meliponiculture)—one of the biggest engineering challenges is **connectivity**. Apiaries are often located in rural areas, deep in the woods, or on farms where 3G/4G signals are weak or non-existent.

If a producer is in the field inspecting a hive, they cannot afford to wait for a loading spinner or lose their management records because the internet dropped. The application must work seamlessly offline and sync data automatically when connectivity is restored.

In this article, I will share the architectural decisions and the technical implementation of the **Offline-First Synchronization Engine** we built for ASFControl using **IndexedDB**, **Dexie.js**, and **React Hooks**.

---

## The Challenge

Traditional web applications rely on a synchronous request-response lifecycle. You submit a form, the app sends a `POST` request to the server, and the UI updates based on the response. If the network fails, the request fails, and the user gets an error message.

For ASFControl, we needed a paradigm shift:
1. **Reads must be instant**, served from a local cache.
2. **Writes must never fail due to network issues**. They should be saved locally and pushed to the server later.
3. **The sync process must be transparent** to the user, recovering automatically when the device comes back online.

## The Architecture: Local Database and Sync Queue

To achieve this in a Next.js / React environment wrapped in a Mobile WebView (Capacitor), we chose **IndexedDB** as our local storage mechanism. Since the native IndexedDB API is notoriously verbose and callback-heavy, we adopted **Dexie.js**, a minimalist wrapper that provides a robust Promise-based API.

Our local database schema is divided into two main areas:
1. **Cached Entities**: Tables for data the user needs to view (e.g., `hives`, `manejos`).
2. **The Sync Queue**: A table that acts as a ledger for all pending mutations (`POST`, `PATCH`, `DELETE`).

### 1. Defining the Local Schema

Here is how we structured our Dexie database. Notice the `syncQueue` table, which is the heart of our offline engine.

```typescript
import Dexie, { type EntityTable } from 'dexie';

export interface SyncQueueItem {
  id?: number;
  url: string;
  method: 'POST' | 'PATCH' | 'PUT' | 'DELETE';
  body: any;
  status: 'pending' | 'failed';
  errorMessage?: string;
  createdAt: string;
}

export class OfflineDatabase extends Dexie {
  hives!: EntityTable<OfflineHive, 'id'>;
  manejos!: EntityTable<OfflineManejo, 'id'>;
  syncQueue!: EntityTable<SyncQueueItem, 'id'>; // The Mutation Ledger

  constructor() {
    super('ASFControlOfflineDB');
    this.version(1).stores({
      hives: 'id, number, status, userId',
      manejos: 'id, date, type, colmeiaId, userId',
      syncQueue: '++id, status, createdAt' // Auto-incremented ID for sequential processing
    });
  }
}

export const db = new OfflineDatabase();
```

When a user registers a new hive inspection while offline, we do not call `fetch()` directly. Instead, we insert a record into the `syncQueue` table and immediately update the local `manejos` table so the UI reflects the change instantly (Optimistic UI update).

### 2. The Synchronization Engine (React Hook)

To orchestrate the sync process, we created a custom React hook: `useSyncData`. This hook is responsible for listening to network state changes and processing the queue.

The core logic operates in two phases:
- **Phase A (Upload):** Process all pending items in the `syncQueue` and send them to the server.
- **Phase B (Download):** Fetch the latest truth from the server and update the local cached tables.

```typescript
"use client";
import { useEffect, useState } from "react";
import { db } from "@/lib/offline/db";

export function useSyncData() {
    const [isSyncing, setIsSyncing] = useState(false);

    useEffect(() => {
        // Trigger sync immediately if the app loads and is online
        if (typeof window !== "undefined" && navigator.onLine) {
            syncData();
        }

        // Attach event listener for when the network recovers
        window.addEventListener("online", syncData);
        return () => window.removeEventListener("online", syncData);
    }, []);

    const syncData = async () => {
        if (isSyncing) return;
        setIsSyncing(true);

        try {
            // PHASE A: Process Sync Queue (Upload Offline Actions)
            const pendingItems = await db.syncQueue
              .where("status")
              .equals("pending")
              .toArray();
            
            if (pendingItems.length > 0) {
                for (const item of pendingItems) {
                    try {
                        const response = await fetch(item.url, {
                            method: item.method,
                            headers: { "Content-Type": "application/json" },
                            body: JSON.stringify(item.body)
                        });

                        if (response.ok) {
                            // Success: Remove from local queue
                            if (item.id) await db.syncQueue.delete(item.id);
                        } else {
                            // Server Error (e.g., Validation): Mark as failed so it doesn't block the queue
                            const errorData = await response.json();
                            if (item.id) {
                                await db.syncQueue.update(item.id, { 
                                    status: 'failed', 
                                    errorMessage: errorData.error || 'Server error' 
                                });
                            }
                        }
                    } catch (e) {
                        // Network Error (Still offline): Break loop or keep as 'pending' for next retry
                        console.error(`Offline Sync failed for item ${item.id}`, e);
                    }
                }
            }

            // PHASE B: Download fresh data from Server
            const hivesRes = await fetch("/api/hives");
            if (hivesRes.ok) {
                const hives = await hivesRes.json();
                
                // Clear existing cache and bulk add new records
                await db.hives.clear();
                
                const offlineHives = hives.map((h: any) => ({
                    id: h.id,
                    number: String(h.number),
                    status: h.status,
                    userId: h.userId,
                    // ... parsing mapping
                }));
                
                await db.hives.bulkAdd(offlineHives);
            }

        } catch (error) {
            console.error("Critical Sync Error:", error);
        } finally {
            setIsSyncing(false);
        }
    };

    return { isSyncing, syncData };
}
```

### 3. Handling Idempotency and Failures

A crucial detail in background syncing is handling failures gracefully. Notice in the code above:
- If `fetch()` throws an exception (e.g., `TypeError: Failed to fetch`), it means the network dropped again. The item remains `pending` and will be retried later.
- If the server responds with a `4xx` or `5xx` status (e.g., validation failed), we update the status to `failed`. This prevents the queue from being perpetually blocked by a bad request (a "poison pill").

By processing the queue sequentially using a `for...of` loop rather than `Promise.all()`, we guarantee that actions are executed on the server in the exact order the user performed them offline, preventing race conditions and maintaining referential integrity.

## Conclusion

Implementing an offline-first architecture requires a shift in how we handle state and side effects. By leveraging **Dexie.js** and the browser's `online` events, we decoupled the user experience from network reliability. 

For the producers using ASFControl, the app feels instantly fast regardless of their location in the field. Data is saved locally, and when they drive back into a coverage area, the system silently pushes their work to the cloud.

Resilient data syncing is not just a nice-to-have feature in agricultural tech—it's a strict requirement for a functional product.
