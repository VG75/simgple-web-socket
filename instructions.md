# Summary of the project

* A serverless, real‑time “duel” platform where authenticated users can see who’s online, send and receive one‑on‑one coding battle invites, and either accept or reject within a timeout window (1 min).
* Upon acceptance, both parties enter a shared session with a synchronized countdown timer (10 min), with the ability for either to stop the duel early.
* Connections and session state are stored in DynamoDB (with TTL for automatic cleanup), routed through API Gateway WebSockets and managed by Lambda handlers.
* Timeout events (no response or end of timer) trigger Lambdas via DynamoDB Streams or SQS Delay Queues to notify clients and tear down sessions.
* A React front end maintains a persistent WebSocket connection (authorized via Cognito), lists online users via a REST lookup of the Connections table, and handles all invite/response/session UI.
* Infra as code (SAM/CloudFormation) defines the WebSocket API, routes, Lambdas, DynamoDB tables, IAM roles, and deployment artifacts for a scalable, maintainable serverless architecture.


## ⚙️ Phase 1 – Infra & Core Lambdas

*Validate your backend primitives before you build any UI.*

1. **Define DynamoDB tables**

   * **Connections**

     * **PK** = `userId` (string)
     * **SK** = `connectionId` (string)
     * **expireAt** (Number, UNIX timestamp) → DynamoDB TTL
   * **Sessions**

     * **PK** = `sessionId` (UUID)
     * Attributes: `userA`, `userB`, `status` (`pending`/`active`/`stopped`), `expireAt` → TTL for auto‑cleanup

2. **Write a CloudFormation/SAM template**

   * WebSocket API with routes:

     * `$connect` → `ConnectLambda`
     * `$disconnect` → `DisconnectLambda`
     * `invite` → `InviteLambda`
     * `response` → `ResponseLambda`
     * `stop` → `StopLambda`
   * DynamoDB tables (with TTL enabled).
   * IAM roles granting each Lambda:

     * `dynamodb:PutItem`, `DeleteItem`, `GetItem` on your tables
     * `execute-api:ManageConnections` on `arn:aws:execute-api:<region>:<acct>:<api-id>/*`

3. **Implement & deploy each Lambda (Node.js)**

   * **ConnectLambda** (`$connect`):

     ```js
     const AWS = require('aws-sdk');
     const ddb = new AWS.DynamoDB.DocumentClient();
     exports.handler = async (evt) => {
       const { connectionId } = evt.requestContext;
       const userId = evt.requestContext.authorizer?.jwt?.claims?.sub;
       await ddb.put({
         TableName: 'Connections',
         Item: { userId, connectionId, expireAt: Math.floor(Date.now()/1000)+3600 }
       }).promise();
       return { statusCode: 200 };
     };
     ```
   * **DisconnectLambda** (`$disconnect`): delete that `connectionId`.
   * **InviteLambda** (`routeKey === 'invite'`): lookup target’s `connectionId` and call:

     ```js
     const apigw = new AWS.ApiGatewayManagementApi({ endpoint: domain });
     await apigw.postToConnection({ ConnectionId: targetConn, Data: JSON.stringify({ type:'invite', from: userId }) }).promise();
     ```
   * **ResponseLambda**:

     * On “accept”, write a new `Sessions` item with `expireAt = now+600`.
     * Broadcast “start” to both.
   * **StopLambda**: on manual `stop` or TTL/SQS trigger, update `status='stopped'` and notify both.

4. **Timer mechanism**

   * **Option A (TTL + Streams)**:

     * Enable Streams on `Sessions`.
     * If a record expires, Streams → a Lambda that does `postToConnection({type:'timeout'})`.
   * **Option B (SQS Delay Queue)**:

     * On “accept,” enqueue a message with `DelaySeconds=600` → consumer Lambda → verify session is still `active` → notify.

5. **Smoke‑test before UI**

   * Deploy with SAM/Serverless.
   * Use `wscat` or a tiny Node script to simulate two `userId` connections and run through the invite/accept/timeout flow end‑to‑end.

---

## 🖥 Phase 2 – Basic UI & WebSocket Client

*Only after your backend “works” in isolation do you scaffold the frontend.*

1. **Scaffold React app**

   * Create a root‑level WebSocket hook (`useWebSocket`) that:

     1. Connects on app load to `wss://…` with the Cognito JWT in query string.
     2. Listens for messages: `invite`, `start`, `timeout`, `stop`.
     3. Exposes `send(route, payload)`.

2. **Online Users list**

   * **How it works**: every client keeps their WS open no matter which page they’re on.
   * When you land on “Invite” page, call a REST endpoint (Lambda+API GW HTTP) that does a `Scan` or (better) a `Query` on `Connections` for TTL > now to get all active `userId`s.
   * Display those in a list; clicking “Invite” calls `send('invite', { to: targetUserId })`.

3. **Invitation UI**

   * On receiving `{type:'invite', from}`, pop up a modal with “Accept”/“Reject.”
   * On click, call `send('response',{ sessionId, accept: true/false })`.

---

## 🛠 Phase 3 – Full Session Flow & Edge Cases

1. **Synchronized timer**

   * Backend “start” message carries `startAt = <UTC ISO timestamp>`.
   * Frontend on both sides does:

     ```js
     const duration = 600000; // 10 min in ms
     const delay = new Date(startAt).getTime() - Date.now();
     setTimeout(() => beginLocalTimer(duration), delay);
     ```

2. **Stop/Cleanup**

   * “Stop” button → `send('stop',{ sessionId })`.
   * StopLambda → broadcasts `{ type:'stop' }`, both clients navigate home.

3. **Idle/No‑response (1 min)**

   * Use the same TTL/Stream or SQS approach with `expireAt = now+60`.
   * On TTL fire, Lambda sends `{type:'no_response'}`.

4. **Multi‑session support**

   * Composite key on `Sessions`: `PK=sessionId`, and GSI on `userId` so you can always list a user’s active sessions if needed.

---


## 📋 Your next Cursor tickets

1. **\[Infra]** Write CFN/SAM for WebSocket API + Routes + IAM + DynamoDB tables (TTL enabled).
2. **\[Lambda]** Implement and unit‑test `ConnectLambda` and `DisconnectLambda`.
3. **\[Lambda]** Implement `InviteLambda` (+ postToConnection snippet from AWS docs) and test with two `wscat` clients.
4. **\[Lambda]** Implement `ResponseLambda` to create `Sessions` with TTL & broadcast “start.”
5. **\[Lambda]** Choose TTL+Streams vs. SQS Delay—implement your 1 min no‑response handler.
6. **\[Lambda]** Implement `StopLambda` to tear down sessions on both manual stop and TTL.
7. **\[UI]** Scaffold React app with a global WebSocket hook and REST call for “getOnlineUsers.”
8. **\[UI]** Build Invite page: list users, send invite, handle incoming invite.
9. **\[UI]** Build Session page: show synchronized timer (use backend’s `startAt`), stop button, handle timeouts.

That sequence lets you prove each piece in isolation, then hook them together. Once you have 1–4 done, we’ll refine the CloudFormation IAM policies and dive into the exact Node.js snippets from AWS docs. Let me know which ticket you want to tackle first.
