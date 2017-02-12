pouchdb-with-service-workers
========

Challenges
----

1. You cannot have long-running tasks inside of a Service Worker, because the browser will just terminate the Service Worker after a timeout. So e.g. you cannot do `db.replicate({live: true, retry: true})`.
2. All replications to and from the local database need to flow through standard ServiceWorker events such as `'sync'` and `'push'`.

Design
---

Basic idea is to use the Service Worker events to trigger single-shot replication events, i.e. to never use `live` or `retry`.

To sync **from** the local database, we can use the `'sync'` event, which fires when the browser goes from an offline state to an online state. [Periodic sync](https://github.com/WICG/BackgroundSync/blob/master/explainer.md#periodic-synchronization-in-design), which allows firing this event on a timer (e.g. every 30 minutes, but only on battery, or only on WiFi), is not currently shipped in any browser.

_Question:_ even if the browser tells us it thinks it's online, the request may still fail. How do we re-schedule new syncs? I guess we need Periodic Sync for this?

To sync **to** the local database, we can use the `'push'` event, which send a push notification from the server to the client via the native push messaging system (GCM/FCM on Android, Mozilla Push for Firefox, WNS for Windows/Edge). 

_Question:_ according to [Mozilla docs](https://developer.mozilla.org/en-US/docs/Web/API/Push_API/Using_the_Push_API#Extra_steps_for_Chrome_support), Chrome only supports `{userVisibleOnly:true}` meaning that you must send a push notification when you receive a push message, which is not very well-suited to PouchDB replication. Also Chrome only accepts GCM/FCM and a token must be supplied in an app manifest file. Has this changed, or is this still the case with Chrome? If so, we might only be able to get this to work on Firefox (and later Edge).

Architecture
---

For the `'sync'` event (replicate from client to server), just call `localDB.replicate.to(remoteDB)` on the event and you're done.

For the `'push'` event (replicate from server to client), it's much more complicated:

1. Register for the `'push'` event
2. Get a URL pointing to the Push Server with an access token
3. Send that URL and that token to the server (Mozilla has a good [sample server](https://github.com/chrisdavidmills/push-api-demo/blob/gh-pages/server.js))
4. Server listens to the CouchDB `_changes` feed
5. On a change, the server sends a push message to the Push Server (debounced to avoid sending too many messages)
6. Eventually the message reaches the client, which essentially says "hey something changed"
7. Client receives the `'push'` event
8. Client calls `localDB.replicate.from(remoteDB)`
