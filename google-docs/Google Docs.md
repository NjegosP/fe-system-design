# Google Docs

## Question
Design a collaborative document editor that allows users to collaborate on a document in real-time.

**Note**: This article focuses on the collaboration aspects of collaborative text editing software and not much about text editing itself. For a deep dive into rich text editing, have a look at the Rich Text Editor system design article.

## Requirements exploration
- Functional
	- Any participant can edit / view the document
	- Updates by peers on the same document are reflected automatically
	- Conflicts are resolved if participants are editing the same part
	- All participants should see the same document revision eventually
	- Offline usage? Not required, but we’ll discuss if there's time
- Non-functional
	- Updates by others are reflected in real-time
	- Support up to 100 concurrent editors ([Google has the same limit](https://support.google.com/docs/answer/2494822))

## Background
Firstly, let's understand what a collaborative editing session over Google docs entails. The following characteristics are observed/required of collaborative editing sessions:

- **Responsive (in the delay sense)**: Actions taken by participants (e.g. inserting/deleting characters, formatting) should be reflected instantly.
- **Real-time updates**: Actions taken by peers should be reflected near instantly.
- **Conflict-prone**: During a session there is a high chance of access conflict as participants work on and modify the same parts of the document, e.g. deleting a sentence when another participant is modifying it.
- **Distributed**: Participants are accessing the web application on their own machines, without any geographical constraints.
- **Ad hoc**: Participants are free to come and go during a session; they may drop off anytime or join anytime.
- **Unpredictable**: In general, participants are not following a pre-planned script, it is not possible to predict what information will be accessed and in what order.

The technical term for such collaborative software is "groupware systems". Most products in Google Workspace (formerly known as G Suite) like Google Sheets and Google Slides also support real-time collaboration.

Note that participants should be considered on the browser tab level. A user can open the same document on two separate tabs. Assuming there is no direct communication between browser tabs for the same user, it will be simpler to treat each tab as a separate participants and sync their document states.

The complexity of real-time collaborative editing solutions arises primarily from communication latency. Ideally, if communication were instantaneous, developing a real-time collaborative editor would be as straightforward as creating a single-user editor as updates from peers would appear as if they were made by the active user.

However, network latency limits communication speed, leading to a fundamental challenge: users expect their own edits to be incorporated into the document immediately. Yet, if these edits are applied instantly, they are necessarily inserted into different revisions of the document due to the delay in communication.

Consider the following example: Alice and Bob start with a document that contains the word "Mary". Bob deletes the letter "M" and then inserts "H", intending to change the word to "Hary". Meanwhile, Alice, without having received Bob's edits, deletes the letter "r" and then deletes "a", with the intention of changing the word to "My". As a result, both Alice and Bob will receive edits that were applied to versions of the document that never existed on their own machines and without any special handling, the document states might diverge.

Therefore the challenge of real-time collaborative editing is to determine how to correctly apply edits from remote users that were originally made in versions of the document that never existed locally and may conflict with the user's own local changes.

Typically, for front end system design, we only need to care about what goes on within the client. However, for collaborative editing, the server also plays an important role in the collaboration protocol, making it necessary to also discuss the server's state and responsibilities.

## Approach
It's hard to design an entire complex system by looking at parts in isolation, but it's also not conducive for learning if the final architecture is presented to you without explaining how we arrived at those decisions.

Hence we will discuss the various aspects of the systems, the various approaches that can be taken, the tradeoffs involved, then make a decision on the overall approach.

It's important to note that some of these decisions we will be making have dependencies on each other and the decisions only make sense when used together as part of the overall approach. So if while reading you're wondering how we know that a certain decision is better, read on first and hopefully the later parts will justify the earlier decisions.

After deciding on the overall approach, we will then dive deeper into the architecture, data model, and APIs to move forward with.

### Rendering and editing rich text
In well-architected systems, rendering the document UI is mostly independent of the communication models and protocols behind the scenes. We explain the "front end" of collaborative editors in more detail in the Rich Text Editor system design article.

Here is a summary of the ways to build rich text editing on the web:

- **DOM with augmented fake cursors**: Using regular DOM containing formatted text using HTML elements and styling augmented with fake cursors. This approach is complex due to cursor height and positioning calculations needed.
- **`contenteditable` attribute**: By adding the `contenteditable` attribute to a DOM element, its content becomes editable and even supports keyboard shortcuts for bold, italics, underline formatting. However, it behaves differently in different browsers and formatting options are limited.
- **HTML `<canvas>` element**: This is an advanced approach that uses a `<canvas>` element and rendering everything within the canvas element – text, layout, cursor, etc. This approach basically sidesteps a lot of what the browser provides, and requires reimplementing everything within a canvas context.

In practice, most rich text editors on the web are built using the `contenteditable` attribute backed by custom event handlers and allows for more content elements.

In 2021, [Google announced that Google Docs will be moving towards a canvas based rendering](https://workspaceupdates.googleblog.com/2021/05/Google-Docs-Canvas-Based-Rendering-Update.html?utm_source=the+new+stack&utm_medium=referral&utm_content=inline-mention&utm_campaign=tns+platform) approach to "improve performance and consistency in how content appears across different platforms". While Google Docs uses a canvas-based approach, it is too complex to discuss and most front end developers do not have much experience with canvas. We recommend being familiar with the `contenteditable` attribute approach first.

In summary, most rich text editors on the web use `contenteditable` and take the following high-level approach:

1. Design a editor-specific model of the document content, usually tree-based
2. Create a mapping between the model and DOM elements
3. Define a set of supported operations on this model (e.g. insert text at position, delete text, format text)
4. Translate user events (keypresses and mouse clicks) into a sequence of these supported operations
5. Update the DOM based on these operations, ideally with minimal DOM manipulation calls

### Request payload contents
When a participant makes an update, or receives an update from others, what should the request payload contain?

The two common approaches are:

- **Payload contains the entire current document**: Sending and receiving the entire current document state.
- **Payload contains only changes (the delta)**: Sending and receiving just the updates made.

### Transmitting entire document
Transmitting the entire document per update has the following properties:

- **Clear document state**: Recipients know the full state of the document the sender was viewing when it was updated.
- **Sender intentions aren't clear**: Just by receiving the updated document, recipients do not know what changes were made unless they compare with their own document revision or the sender's previous document revision, both of which are troublesome to compute.
- **Not scalable**: As the document grows, the amount of data to transmit will increase proportionately. For large documents, each request will take longer to transmit.

### Transmitting only changes
Transmitting just the changes has the following properties:

- **Small request payload size**: Since only the changes are sent, the request payload size is generally small, typically containing only the command (e.g. insertion, delete, formatting) and associated metadata (e.g. characters and position to insert/delete).
- **Efficient request payload size**: The document's length has no impact on the request payload size since the changes are not reliant on the current document state.
- **No document state information**: It's not clear which version of the document the changes were made against. This can be mitigated by including a document revision number in the request payload – more on that later.

### What should requests contain?
A short response time (under 500ms) is required for a real-time WYSIWIS (What-You-See-Is-What-I-See) experience. Meaning when a user makes an edit, all other participants should see that change reflected on their screens within 500ms. If participants see slightly different or outdated versions of the document, the cohesiveness of the session is ruined.

Updates made by participants should be communicated in a lightweight fashion, thus the request payload for updates should ideally **contain just the changes**.

### Network architecture / communication model
There are two common models for network communication between groupware system participants:

- **Client-server model**: Central server; all participants communicate only with the server
- **Peer-to-peer**: No central server; all participants act as both a server and client

#### Client-server model
In a client-server network model, participants (clients) request resources from a central server and send payloads to it. The server processes these requests and provides the appropriate responses.

![ecf04e23407a75a16150093f37019a12.png](../_resources/ecf04e23407a75a16150093f37019a12.png)

#### Advantages:

- **Centralized source of truth**: All updates have to first go through the server. The server determines the latest document revision and holds the source of truth.
- **Robustness**: When new participants join or one of the existing clients crash (due to bugs, network flakiness, etc), the latest document can be fetched from the server.

#### Disadvantages:

- **Single point of failure**: The central server is a critical component; if it fails, the entire network can be disrupted.
- **Cost**: Higher infrastructure and maintenance costs due to the need for powerful central servers.
- **Scalability**: Can become a bottleneck if the number of clients grows rapidly without proper scaling.

### Peer-to-peer (P2P) model
In a P2P network model, each participant acts as both a client and a server. Peers communicate directly with each other without the need for a central server.

![c1f718cd71e665caad97772889679973.png](../_resources/c1f718cd71e665caad97772889679973.png)

#### Advantages:

- **Lower latency**: Each participant communicates with each other directly without the need for a middleman, resulting in lower latency in general.
- **Cost effective**: Without a central server, infrastructure costs are lower.
- **Load distribution**: When more participants join, the extra network load is distributed among all participants, preventing bottlenecks.

#### Disadvantages:

- **Synchronization problems**: Without a central authority, it is hard for participants to determine if their version of the document is the latest version and what updates are missing. Each participant needs to keep track of the peers' document states and ensure they are up-to-date.
- **Complicated consensus**: When new participants join or one of the existing clients crash (due to bugs, network flakiness, etc), the new member has to figure out how to fetch the latest document revision among all the peers.
- **Server is still needed**: Since the document needs to be persisted to a database, a server is needed anyway.
- **Security**: If a participant goes rogue, all connected participants will be affected.

### Which model to use?
Although the client-server model seemingly has fewer advantages, the source of truth being on the server is an important functional property that makes or breaks the entire system:

- **Database persistence**: The document state has to be persisted in a database. Without a central server to save the document frequently, updates could be lost if clients disconnect before saving the document to the database.
- **Source of truth**: Since new participants are able to join at any time, having the source of truth on the server simplifies the logic of reliably obtaining the latest document – new participants can fetch the latest document from a single consistent location, the server.
- **Optimize for reliability and performance**: Reliability and performance are the most important factors for a seamless collaboration experience and making sure all participants are updated. In practice, most sessions will not see a large number of participants (> 20) at any given time; there will only be a few participants at most. We should optimize for reliability and performance rather than server costs.

With these reasons in mind, a client-server model is preferred. P2P models are more suitable for applications like video chat, where request losses can be tolerated, and not all the data has to be persisted.

### Concurrency control model
There are two kinds of operations that need to be made when discussing collaborative editors:

- **Local updates**: Updates made to the document by the user viewing it.
- **Merge updates from peers**: Updates made to the document by peers.

All these updates have to be merged into the current document. Conflicts might arise if two users are editing the same section of the document (e.g. somebody deletes a sentence when another person is adding a word to the sentence).

Concurrency control is the activity of coordinating interfering actions that operate in parallel to maintain consistency between participants and resolving conflicts that arise between participants.

They can be broadly classified into optimistic and pessimistic types:

- **Pessimistic**: Pessimistic methods guarantee that no conflicts occur. Their main objective is inconsistency avoidance. They require communication with other sites or a central coordinator before any data changes are made. This communication can be explicit (e.g. floor control policy), or implicit, where the user's program handles it in the background (e.g. locking). In general, pessimistic methods do not require conflict resolutions but have higher network latency and longer response times.
- **Optimistic**: Optimistic methods do not bother with avoiding conflicting updates. They require no prior communication before making local changes and are well-suited for high-latency communications channels since the results of a user's actions can be displayed without waiting for a communications round-trip. The user applies the changes immediately and updates the server which then notifies the peers. If multiple participants make simultaneous changes, a conflict resolution algorithm creates compensating changes to ensure everyone reaches the same final state. Optimistic methods have zero/near-zero local response times but may require conflict resolutions.

Text editing is unlike traditional form submission, where users can tolerate updates taking up to a few seconds to complete. Textual updates made by the user should be reflected instantly without any latency. To support zero latency local updates, clients should maintain a replica of the document state locally and user updates are made against the replica so that they can be reflected on their UI instantly.

Let's have a look at the various concurrency control mechanisms, their properties, and their pros and cons.

### Last write wins model
For traditional web applications like admin dashboards. If two users modify the same entity simultaneously (e.g., changing a person's name), a race condition occurs. The final name saved in the database will be the one submitted by the request that arrives last.

In distributed computing, this behavior is called "last write wins". Obviously, "last write wins" will not work for a collaborative editor, not at the document level at least. It is hardly considered collaborative at all!

### Floor control model
"Floor control" is a protocol which determines which user has control (has the floor) and how to take turns when multiple people access a resource, a document in this case.

- **Token-based**: A token circulates among participants, and only the participant holding the token can make changes. Participants can request for a token and the predefined policy (e.g. first-come-first-served, or free-for-all) determines if the request is granted.
- **Chair-controlled**: A designated chairperson or moderator grants and revokes participant access to resources.

Participants without control can still see the updates being made in a live fashion.

Floor control helps to prevent conflicts and ensure orderly interactions. Evidently, this method does not fulfill the requirements because only one participant can make edits at any one time; peers have to wait for their turn to edit. One way around this is to allow participants to take over control of the token anytime they want, but it's not a great user experience for participants who are still typing to have control taken away from them suddenly.

Locks model
Locking is the concept of preventing unwanted access to data (read and/or write). In the context of collaborative editing, when an editor starts editing a document, the document (or parts of it) can be locked to prevent unwanted modifications from peers.

There are a few issues to discuss related to locking:

- **Locking granularity**: Locking can be done on the document level, paragraph level, sentence level, word level, etc. Document-level locking is essentially the "floor control" mechanism. Finegrain granularity is better, but what should happen in sentence-level locking when the paragraph containing the sentence is being deleted?
- **Lock requests**: Locks can be requested in both optimistic and pessimistic fashions.
	- **Pessimistic locking**: Clients make a network request to obtain the lock. Clients will only start allowing modifications after the lock has been granted. There is network latency involved in acquiring the lock, so users will experience a delay before being able to do any actions, which can be annoying.
	- **Optimistic locking**: Clients allow modifications and requests for the lock at the same time. If the request for the lock was rejected (possibly because another person also requested for the lock at the same time), the client will roll back the updates prior to requesting for the lock.
- **Lock releases**: After a lock is acquired, when should it be released? Should the lock be requested when the cursor is moved or when the key is struck? For example, if locks are released when the cursor is moved then a user could move to one place to copy some text only to be locked out from pasting it into their previous location. On the other hand, if locks are retained too generously, users might still be granted the lock even when they no longer need it, e.g. a lock is redundant when the user leaves the document open but goes away from the keyboard. Determining the idle threshold is tricky. Similar to floor control, allowing users to take over control of the lock anytime they want is one way of mitigating lock releasing issues.
Locking is an improvement over floor control (which is also technically a type of locking) and is viable at the right granularity with optimistic locking and appropriate lock releasing strategies due to the ease of implementation. In fact, collaborative editing apps like Quip used a lock-based concurrency model in earlier versions of their product. Locking is actually a viable approach for real-time collaboration if the likelihood for editing the same section is low.

### Transaction-based model
A transaction-based approach is an optimistic method, where users make changes locally and changes are validated at the end of the transaction, similar to distributed version control systems like Git and Mercurial. Version control systems manage changes to documents or resources by maintaining different versions and allowing users to merge changes.

Each user has a copy of the document within their computer and can make their changes locally, without any locks or other restrictions. Changes are not pushed immediately. When the user is done updating, the changes are committed and pushed up to the server. If there are any conflicts, the transactions are rolled back and the user has to resolve them in order to update the server.

This approach has the benefit of zero latency local updates and all participants can update their documents simultaneously without any locking restrictions. However, in this approach, updates aren't real-time – users are required to explicitly push their updates and explicitly pull updates from others. Additionally, it requires users to resolve conflicts themselves, which is annoying and unreasonable to expect of common users.

### Version-detection model
In a version-detection model, the document state is replicated in each user's machine and participants can make changes locally, which are then propagated to peers as soon as possible. Under good network connections, users should see updates from peers in under a second.

Each update request contains the new data and the document revision the change was acted on. When the server receives the update request, it first checks if the revision in the request is the same as the current revision. If the revisions match, the server updates the document, saves it as a new revision, and broadcasts this information to all peers so that everyone is on the latest revision. If they are different, then there's a "version mismatch" which can be resolved in one of the following ways:

- **Reject the request**: This avoids having to do any conflict resolution but isn't ideal as the frequency of it happening can be quite often when there are multiple participants simultaneously updating the same part of the document.
- **Compensate changes**: Automatically compensate the changes to rectify the version mismatch and bring the system to a consistent state. In most cases, compensating is straightforward if changes were made to different parts of the document (probably no need to compensate at all!). If the changes do conflict, we can use conflict resolution methods like Operational Transformations (OTs). On the other hand, if Conflict-free Replicated Data Types (CRDTs) are used, there is no need for compensation.

The version-detection model with compensation enjoys the benefits of zero latency local updates, all participants being able to update their documents simultaneously in real-time without any locking restrictions, and will eventually converge into the same version of the document if there's a robust conflict detection and resolution approach.

Version detection trumps the transaction-based model and fulfills all the real-time collaboration requirements. The only issue being – how exactly are conflicts resolved? We'll explore them in more detail below.

### Which concurrency model to use?
Only the version-detection consistency model is able to meet all the real-time collaborative editing requirements because it has the following properties:

- **Replicated**: The document is replicated in the machines of all participants. This is necessary for the model to make optimistic updates.
- **Optimistic and non-blocking**: Updates are made locally, without worrying about conflicts. Optimistic behavior is necessary for zero-latency responsiveness during local updates – the speed or reliability of your network connection will not influence how fast users can type.
- **No locking**: The entire document is available to every participant all the time.
- **Automatic conflict resolution**: Any detected conflicts are resolved automatically on the server. Possible approaches include creating compensating changes and informing the client (OT), or using data structures that resolve conflicts automatically (CRDTs).

From each participant's perspective, it will feel as if they are editing an offline Word document – there is no latency involved when editing.

### Conflict resolution approaches
Conflicts can arise when two or more participants are editing the same part of the document. Let's run through an example.

Assume we have a document containing a single word "ABCDE", and Alice and Bob edit it at the same time:

- Alice deletes the fourth character "D". Her computer sends a request `DEL @3` to indicate deletion of the fourth character, which is at index 3 (zero-based indexing).
- Bob deletes the second character "B". His computer sends a request `DEL @1` to indicate deletion of the second character, which is at index 1 (zero-based indexing).
The server will process both requests. For now, the server simply executes the commands and relays them without any special handling). The intended end result after both deletions is "ACE".

Since network latency is unpredictable, one of two scenarios can happen:

![14dfaa35bb6f5004c3a759efc58ee185.png](../_resources/14dfaa35bb6f5004c3a759efc58ee185.png)

1. **Alice first**: The server receives Alice's request before Bob's, resulting in the server deleting the fourth character then the second character "ABCDE" -> "ABCE" -> "ACE". In this scenario, the document on the server correctly deletes Alice's and Bob's intended character. The server ends up with "ACE".
2. **Bob first**: The server receives Bob's request before Alice's, resulting in the server deleting the second character then the fourth character: "ABCDE" -> "ACDE" -> "ACD". Notice that when the server processes Alice's request, the fourth character of the document is now "E", but Alice wanted to delete "D". The server ends up with "ACD".

In this naive approach, depending on whose request the server receives first, the server ends up with different document states.

In fact, Bob will always end up with "ACD" regardless of whose request reaches the server first. Remember that updates are first made locally then sent to the server to be broadcasted to others. The problem is that offsets depend on the state of the document at the time an edit was made. Request payloads include the offsets but not the context they depend on, which results in the divergence.

This is just one of the possible scenarios that result in conflicts. There are other possible combinations like insertion + insertion, deletion + insertion, deletion + deletion, etc.

### Conflict resolution properties
Therefore we need a conflict resolution approach that adheres to these properties as much as possible:

- **Convergence**: All replicas will eventually reach the same state, provided that they have received and applied the same set of operations (the quiescence state).
- **Causality preservation**: Ensures that the order of operations respects the causal relationships between them. For example, if one operation logically follows another, the system must apply these operations in that order to maintain consistency (e.g. deletion of a character after its insertion).
- **Intention preservation**: Ensure that the original intent of an operation is maintained after concurrent operations are merged. This means that the result of merging concurrent operations should align with what users expect their operations to achieve, even in the presence of conflicts. E.g. Alice makes the entire sentence bold and Bob adds a word to the sentence at the same time, the end result should be that the sentence including Bob's word is bold.

Two common conflict resolution approaches are used:

- **Operational Transformations (OT)**: OT accounts for context at the point of editing and transforms the operations accordingly (e.g. by modifying the offsets of insertions/deletions).
- **Conflict-free Replicated Data Types (CRDTs)**: CRDTs enforce the use of data structures where updates are commutative and associative such that the order of operations does not matter.

**Note**: Explaining each conflict resolution approach in detail is beyond the scope of this article (there isn't enough time during interviews anyway). However, you should be able to explain the general principles using an example. We will explain how each approach works, provide examples and links to resources for your further reading.

### Operational Transformations (OT)
OT was originally developed for collaborative editing of text documents, allowing multiple users to edit simultaneously without conflicts. It maintains the consistency of document states across different clients by transforming operations based on the context of other concurrent operations.

OT systems typically use a replicated document storage model, where each client maintains a local copy of the document. Users operate on their local copies, and changes are propagated to other clients. When a client receives changes from another client, it applies transformation functions to ensure that the local document remains consistent with the updates.

The key components of OT include:

- **Operations**: These are the basic actions users perform, such as inserting, deleting, or modifying text. Each operation is associated with a position in the document.
- **Transformation functions**: These functions determine how to adjust operations when they conflict. For example, if two users insert text at the same position, the transformation function will resolve the conflict to maintain a consistent document state.
- **Control algorithms**: These algorithms manage the sequence and context of transformations. They decide which operations should be transformed against others based on their causal relationships, ensuring that the order of operations respects the intent of each user.

Let's revisit the Alice and Bob example and see how applying OT can solve the issue.

![7579e65ac685f2220000b6c0be6f3fed.png](../_resources/7579e65ac685f2220000b6c0be6f3fed.png)

1. Alice before Bob: The server receives Alice's request before Bob's, resulting in the server deleting the fourth character then the second character "ABCDE" -> "ABCE" -> "ACE". In this scenario, the document on the server correctly deletes Alice's and Bob's intended characters. However, Bob's computer realizes that Alice's change was made before Bob's deletion, hence it transformed Alice's change from `DEL @3` -> `DEL @2` since Bob deleted an earlier character. Bob ends up with "ACE".
2. Bob before Alice: The server receives Bob's request before Alice's, resulting in the server deleting the second character: "ABCDE" -> "ACDE". When the server receives Alice's request, it realizes that Alice's change was made before Bob's deletion, hence it transforms Alice's change from `DEL @3` -> `DEL @2` to account for Bob's deletion of an earlier character. The server correctly deletes "D" (the third character) and passes this transformed operation to Bob. Both the server and Bob end up with "ACE".

Both servers and clients can perform OT and must handle all the various ways that insertion, deletion, and formatting changes can be paired and transformed against each other. The example above showed how a deletion is transformed against a deletion. Some other examples of transformations:

- Formatting is expanded when transformed against insertions: `FORMAT BOLD @10-20 `transformed against `INS "ABC" @15` results in `FORMAT BOLD @10-23`.
- Not all changes will conflict and in those cases do not require transformation. E.g. F`ORMAT BOLD @10-20` and `FORMAT ITALIC @15-25` do not conflict as text can be both bold and italic at the same time.

How does the server know whether Alice's and Bob's requests were made with or without accounting for the other party's changes and whether it has to do any transformation? Each time an update is made on the server and the document is modified, the document is saved as a new revision (e.g. using timestamps or monotonically increasing integers). Timestamps are not a great choice because machine time can be manipulated which results in an incorrect order when sorting by time, a [monotonically](https://en.wikipedia.org/wiki/Monotonic_function) increasing positive integer is preferred.

Requests and responses can include the document revision number so that both servers and clients know the document version the other party was seeing when the request was made and can correctly determine if operations require transformation.

**Analysis of OT**: Let's take a look at how OT fulfills the conflict resolution properties:

- **Convergence**: OT uses transformation functions to modify operations so that they can be applied consistently across all replicas, even if they arrive in different orders. The core idea is that if two operations conflict, the transformation functions adjust one or both operations to ensure they can be applied in any order but still lead to the same final state.
- **Causality preservation**: OT systems often use causal history tracking to ensure that operations are applied in an order that respects their causal dependencies. This is usually done by tagging operations with metadata, such as timestamps or revision numbers, which indicate their causal relationships.
- **Intention preservation**: OT payloads commonly include the command intention and context (document revision number). The transformation functions consider the context in which an operation was originally applied. The command along with the contextual awareness helps in preserving the original intention even when the document has changed due to other concurrent operations.

OT works well for text-based documents but can be less effective for other types of data structures, such as hierarchical or non-linear data. Implementing OT for these types of data requires additional effort and customizations.

There are many implementations of OT algorithms and various consistency models, which are beyond the scope of this article. If you're interested, check out the following links:

- [What’s different about the new Google Docs: Conflict resolution](https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs_22.html)
- [Operational Transformations as an algorithm for automatic conflict resolution](https://medium.com/coinmonks/operational-transformations-as-an-algorithm-for-automatic-conflict-resolution-3bf8920ea447)
- [Visualization of OT with a central server](https://operational-transformation.github.io/)

### Conflict-free Replicated Data Types (CRDTs)
If you were wondering – why use offset indices for the updates when they are highly coupled to the current document state and go through the hassle of resolving conflicts with complex algorithms like OT? Why not use a different payload or data structure that provides more information regarding the update and makes merging updates easier? That is exactly Conflict-free Replicated Data Types (CRDTs) aims to do.

CRDTs are advanced data structures designed for distributed systems, enabling multiple users or applications to update shared data concurrently without coordination which eventually converge to the same state (strong eventual consistency) when all updates have been received and applied. Instead of resolving conflicts using OT, CRDTs are built so that the operations performed on the data are inherently conflict-free or can be automatically resolved in a consistent manner.

CRDTs have the following properties:

1. **Concurrent updates**: CRDTs enable independent updates across multiple replicas of data. Each replica can be modified without needing to coordinate with others, making them ideal for environments where network connectivity may be intermittent. Updates can be propagated using the gossip protocol without the need for a central authority.
2. **Monotonic increasing updates**: Updates to a CRDT must be monotonic, ensuring that new values are always greater than or distinct from previous values. This allows for a clear progression of state changes.
3. **Commutative and associative operations**: The operations in a CRDT must be commutative (Order does not matter -> `A + B + C === C + B + A`), associative (Grouping does not matter -> `(A + B) + C === A + (B + C)`). This ensures that all replicas, even if they receive operations in different orders or merge states at different times, will end up in the same state.
4. **Automatic conflict resolution**: CRDTs incorporate predefined algorithms (e.g. last writer wins) that automatically resolve inconsistencies that may arise from concurrent updates. This means that even if different replicas make conflicting changes, the CRDT can merge these changes without manual user intervention.
5. **Eventual consistency**: Although replicas may have different states at any point in time, CRDTs guarantee that all replicas will eventually converge to the same final state once all updates have been propagated, regardless of the order in which these updates are received. This is often referred to as "strong eventual consistency", which ensures that no inconsistent states are held.

The CRDT model is somewhat similar to Git – every developer in the organization possesses a copy of the repository and is allowed to make changes locally. At the end of the day, the developers can merge changes with every other developer however they like: pair-wise, round-robin, or through a central repository. Once all the merges are complete, every developer will have the same state. However unlike Git, a CRDT prescribes a way to merge conflicts automatically and can merge out-of-order changes.

**Example of CRDT – Grow-only set**: An example of a CRDT is a grow-only set. A grow-only set is an unordered set that only allows addition of unique elements. It is a CRDT because:

- The set can be replicated.
- Each replica can add any element it likes and the addition is a monotonically increasing update.
- Each replica can be merged back together in any order.
- Once all merges are complete, all replicas will have the same contents (the union of all individual sets).

**Representing text in CRDTs**: Collaborative text documents can be represented using sequence CRDTs like lists (e.g. [Linear Sequences and Replicated Growable Array](https://www.bartoszsypytkowski.com/operation-based-crdts-arrays-1/)) and trees. Unsurprisingly, text CRDTs are more complicated to implement than a grow-only set. [Cola](https://nomad.foo/blog/cola) is a text CRDT written in Rust, but the data structure can be implemented in any language.

Text editing also involves deleting. How can we represent deletion in CRDTs? The trick is to track deletions as well by using two separate grow-only data structures, one to track insertion and another to track deletion (known as tombstone markers). The resulting value is the items in the insertion set minus the items in the deletion set. Complex CRDTs are often combined from smaller CRDTs which helps tremendously in preserving the CRDT properties.

**Analysis of CRDTs**: Let's take a look at how CRDTs fulfill the conflict resolution properties:

- **Convergence**: CRDT operations are designed to be commutative and associative, meaning the order and grouping in which operations are applied does not affect the final state. This is a key reason why all replicas converge to the same state.
- **Causality preservation**: Operations are applied to other replicas in a way that respects causality. For example, operations can be propagated in any order, but an operation might be buffered until all preceding causally related operations have been applied. E.g. a deletion only takes effect until the insertion has taken place. In a Last-Writer-Wins Register (LWW-Register), updates are tagged with timestamps, ensuring that the most recent update (according to causality) prevails.
- **Intention preservation**: CRDTs incorporate predefined conflict resolution strategies that aim to preserve user intent. These strategies are typically application-specific and ensure that the merged state reflects the combined intentions of all concurrent operations. The specific semantics of CRDT operations are crafted to ensure that, when two operations conflict, the resolution preserves the most meaningful aspects of each operation. However, there's some amount of subjectivity and is highly implementation and use-case dependent.

**Drawbacks of CRDTs**: While CRDTs are more modern compared to OT, it does come with some drawbacks:

- **Metadata overhead**: CRDTs often require additional metadata to track operations, revisions, or unique identifiers. This metadata can grow over time, leading to increased storage requirements, especially in large-scale systems or with complex data types.
- **Ever-increasing size**: CRDTs have a monotonically increasing state, often having to track removals that do not appear in the final visual result. This means the data will only grow over time. Garbage collection or cleanup mechanisms can be used but they can be technically challenging to implement without causing inconsistencies in replicas.
- **Conflict resolution**: While CRDTs are designed to resolve conflicts by merging concurrent updates in a predefined way, this automatic conflict resolution might not always align with the desired application semantics, leading to unexpected results.

CRDTs entirely bypass the need for causality preservation as updates can be merged in any order and still converge into the same ending state. Whether CRDTs can preserve intention depends on the chosen conflict resolution strategy and implementation.

### Which conflict resolution approach to use?
Both OT and CRDTs are designed to manage concurrent updates to shared data in a way that maintains consistency across all replicas, but they do so using different methodologies and with varying trade-offs.

**Convergence**: OT can guarantee convergence but requires more careful handling of operation order and context. It often requires a central server or a more tightly coordinated communication protocol to ensure consistency. CRDTs are designed to guarantee eventual consistency between replicas and will converge to the same state as long as they receive all updates. CRDTs is superior here because it has stronger convergence guarantees.

**Technical complexity**: Implementing CRDTs for complex or hierarchical data structures can be challenging, requiring careful design to ensure that the operations maintain the desired properties. Implementing OT is also complex, especially when designing the transformation functions that must handle all possible conflicts and concurrent operations. Although CRDTs' consistency model is easier to understand, it's harder to understand and implement CRDTs for text structures. In the context of text editing, OT takes the lead.

**Ecosystem**: OT is a mature technology and has been implemented in several well-known collaborative editing systems with many mature libraries and tools available – Google Docs itself uses OT. CRDTs is the new kid on the block, but has been well-studied over the years and many libraries implementing CRDTs have been created. Figma, a collaborative design editing software uses CRDTs. There is no clear winner in terms of ecosystem as both approaches have been well-studied (and criticized).

Overall, there's no clearly superior choice. Both CRDTs and OT can be used to implement collaborative editors. CRDTs are general purpose while OT has its roots in document editing. While CRDT is newer and more trendy, OT is mature and excels in real-time collaborative editing applications where low-latency, immediate feedback, and fine-grained control over user intentions are critical.

The rest of the article assumes usage of OT as the conflict resolution approach. The primary reason is that Google Docs itself is implemented using OT, so there are also more resources in terms of implementing collaborative text editors using OT as opposed to CRDTs. It is also easier to explain how OT works as opposed to text CRDTs, which has very complex structure.

### Collaboration protocol
We have discussed the conflict resolution approaches, but it is only one part of the story. There are other unanswered questions related to collaborative editing using OT:

- **Request scheduling**: When are update requests sent? On every keystroke, after the user has stopped typing, or something else? If a user makes multiple back-to-back updates, should they all be sent to the server as they are made or is there a better way to schedule them?
- **Multiple participants**: The OT examples above demonstrate how transformations work for two participants. What happens when there are more than two participants in the session and operations have to be transformed against multiple peers?
- **Participants joining**: How do users who join an editing session mid-way start updating their replica and receive updates from others?

A collaboration protocol can be designed to answer these questions. It might be helpful to try out this [interactive visualization of OT with a central server](https://operational-transformation.github.io/) before reading on.

#### Central server
While OT does not require a central server at its core, having a central server simplifies certain things.

A central server architecture makes it simple for clients to stay in-sync with the server. The role of the server is to receive updates and serve as the authority of the document state, transforming operations when clients send requests that conflict with the latest document. The server can also reject operations if the payload is invalid or if the operation was on a document revision that is too old, the operation is too hard to transform, or there are too many operations to transform.

When the server receives an update request, it will broadcast the update to connected clients. Clients do not need to care about how many other connected clients there are. It relies on the server to inform itself of the changes to make. From a client's perspective, having N peer clients (where N is the number of peers) take turns every second to make updates in a round robin fashion is equivalent to a single peer client making updates every second. To a client, having one peer is the same as having ten peers. This allows us to achieve N-way synchronization by running N independent two-party synchronization between each client-server pair. Clients only have to focus on synchronization with the server, and servers can treat all clients equally; there are no special clients to consider.

Participants who join mid-way will be served the version of the document at request time. From that point on, they are a connected client and can send and receive updates like any other participant, as long as the updates are received in order after downloading the initial document.

### Store document as a revision log
The document can be stored as an append-only log of operations/changes similar to the [Event Sourcing pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing). Any revision of the document can be reconstructed by replaying the operations from the start up till that point. Hence the latest revision of the document can be obtained by replaying all the operations since the start. This method of storing data also allows clients to view the document's version history.

![b8e0e96a873f63b49594637dfc2aa27a.png](../_resources/b8e0e96a873f63b49594637dfc2aa27a.png)

Note that the document state is not stored in each log entry as it can be derived from the preceding operations.

Storing as a log is necessary since some clients can be extremely outdated and need to catch up with the current document. Imagine the scenario where a user disconnected from a collaboration session and left the tab open. When they reconnect the next day and others have made updates since then, upon reconnecting, the client sends their current outdated document revision number which allows the server to determine the granular changes since that document revision and respond with a list of operations to be performed on the client, which might also need transformation.

Clients that join the editing session of a non-empty document do not need to fetch the entire log from the start; the server responds with the current state of the document. Subsequent document revisions can be computed via applying new operations (both local and from peers) to the initial session document state.

It's not demonstrated above, but each log entry can contain multiple operations as well.

### When to send update requests
When should update requests be sent? Should update requests be sent on every keystroke, after the user has stopped typing, or something else?

- **Keystroke**: Send a request per keystroke. Not great because it can be taxing on the server and many potentially redundant operation logs created.
- **Debouncing**: Send a request after the user has stopped typing for a short duration (e.g. 300ms). Might not be ideal for users who type a lot without stopping as they can lose data if their browser crashes before the updates are sent out.
- **Throttling**: Send a request every fixed interval (e.g. 300ms) during continuous typing. Seems viable but it's tricky to determine the throttle duration value.

None of the above strategies are truly viable because they all allow a user to send requests in parallel, meaning multiple requests are in-flight at the same time. This is a problem because the server is not guaranteed to receive the requests in the order they were sent, which can lead to race conditions and impossible operations.

There are ways to resolve out-of-order requests, but they aren't trivial. The client can include a monotonically increasing integer for every request in the session but the server has to keep track of the last request sent so far and if it receives an out-of-order request, defer the future request, then wait for the missing request (buffering future operations).

An elegant way taken by Google Docs is to ensure that each user only has a maximum of one update request in-flight (aka pending) by using a local updates buffer (a queue). The local updates buffer is cleared and operations in the buffer are sent to the server, under one of the following scenarios:

- If there are no pending requests, after a short idle duration (~200ms)
- If there is a pending request, when the pending request has returned

When a user starts typing:

1. User operations are reflected locally on the client computer and added into the local updates buffer
2. After a short duration, a request is made which includes the operations in the local updates buffer
3. The local updates buffer is cleared upon sending of the request and any new local updates made while there's a pending request are added to the local updates buffer
4. If the local updates buffer is not empty, it only sends the next request after it receives a response for the current request from the server, even if the idle duration has passed
5. This repeats until the local updates buffer is empty (when the user has stopped typing) and there are no more operations to send to the server

**Fast vs slow connections**: Since there can only be one pending update request, the frequency of requests is highly dependent on the client's connection speed.

![a6207bfbb2d6d108dbbb90add1d82b2f.png](../_resources/a6207bfbb2d6d108dbbb90add1d82b2f.png)

- **Fast connection**: Clients with faster network connection speed will send requests and receive responses faster, resulting in more frequent requests and each request containing a smaller payload. The local updates buffer will be cleared more often.
- **Slow connection**: Clients with a slower network connection speed will see fewer requests and each request will contain a larger payload. The local updates buffer will be cleared less often.

By using a local updates buffer to hold local operations and having only one pending request per client, clients ensure that operations are sent in order and the server can immediately process a client request upon receiving it, knowing that every request from a client is the latest one.

### Operation granularity
A suitable operation granularity is one which doesn't result in too many operations yet also allows differentiation between the intentions. Consider the following scenarios:

- If a user is typing continuously, it'd be more efficient to make a single insertion operation that contains multiple characters rather than one insertion operation per character.
- If a user holds backspace and deletes multiple characters, it'd be more efficient to make a single deletion operation that contains the deleted characters rather than one deletion operation per character.
- If a user types some characters, realizes there's a typo, deletes the erroneous characters, then types again, these actions constitute "continuous typing" and could be coalesced into a single insertion operation. In this case, when the operations are sent to the server, the server does not see any deletions at all.

The following guidelines are chosen:

- **Coalescing of continuous ranges**: Operations on continuous ranges can be combined/coalesced into a single operation of the same kind.
- **One operation per intention**: Each intention (Insertion, Deletion, Formatting) will be a different kind of operation unless they are part of continuous typing.
- **Coalescing of continuous typing actions**: Continuous typing events include forward typing, deleting backwards, and deleting forwards.

### Update request payload

It wasn't explicitly mentioned, but each update payload can contain multiple operations. On slower network connections where requests round-trips take longer, there is a larger window for users to make local updates and sometimes that can include different kinds of operations, not just insertions.

In earlier network diagrams, the update requests only displayed single operations, but requests on Google Docs actually include an array of operations, which is all the operations in the updates buffer. The server iterates through the array of operations in order and transforms them where necessary.

By allowing update requests to contain an array of operations, the buffer is cleared more frequently and the likelihood of losing unsent changes (possibly due to crashes or closing of tabs) is lower.

### Putting them together
We've explained above how a client-server architecture can scale well for a large number of participants. Therefore we can focus on the communication between clients and servers and what information each tracks.

Each client tracks these information:

1. Latest document revision: The identifier of the most recent revision sent from the server to the client. Google Docs uses a monotonically increasing integer as the document revision id/number. This value is included in the request payload and response of each update request.
2. Local updates buffer: Changes (operations) that have been made locally and not yet sent to the server. This is the updates buffer explained earlier.
3. Sent updates buffer: Changes (operations) that have been made locally, sent to the server, but not yet acknowledged by the server. Tracking sent operations is necessary because requests can fail and the client should retry the request if a request fails.
4. Updates log: Committed changes (operations) for the document since the initialized document revision. These changes could be the user's updates that have been acknowledged, or updates by peers that have been pushed to the client by the server. The operations can be already transformed on the server, or transformed on the client depending on the order of updates.
5. Initial document state: The state of the document when the client first joined the editing session.
6. Document state: The current state of the document displayed on the client. This can be computed from the initial document state, the received updates, updates buffer, and pending updates.

Note that the operations in both the local updates buffer and pending updates buffer can be transformed depending on the contents of the received updates from the server. It is the client's responsibility to:

- Track the statuses of the local updates – send them to the server and move them to the sent updates buffer when appropriate.
- After the sent updates have been acknowledged, move the sent updates to the updates log.
- Receive updates from the server and transform any local updates where relevant.
- Combine initial document state with updates log to compute the latest document state and display it.

The server contains the following information:

1. Pending updates queue: A list of all changes (operations) received from clients that have not been processed.
2. Revision log: A list of processed changes that gives the complete history of the document.
3. Document state: The current state of the document as of the last processed change. This value can be derived by replaying all the changes since the start but is computed and cached so that it can be instantly sent to new clients that join the session. It is recomputed whenever a new change is processed.

The server's responsibilities are to:

- Send clients the latest state of the document when they join the editing session.
- Transform updates by clients if they were originally made on outdated revisions.
- Broadcast updates to the other clients.

*Source: [What’s different about the new Google Docs: Making collaboration fast](https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs.html)*

Let's run through a practical example of how the server works with clients to enable real-time collaborative editing using operational transformations.

![9ca9a1e64d1df78a00e8ac289cc2c3ac.png](../_resources/9ca9a1e64d1df78a00e8ac289cc2c3ac.png)

1. Alice, Bob, and the server start with an empty document
2. Alice types "Hello" and the insertion operation is added into her "Local updates". Alice sees the "Hello" instantly within her document replica
3. Alice's insertion of "Hello" is sent to the server and moved from "Local updates" to "Sent updates"
4. The server receives the request and adds Alice's insertion operation into its "Pending updates" queue
5. At the same time, Alice types the characters " world". This insertion is added to "Local updates" but will not be sent to the server until the "Sent updates" is empty
6. Server processes Alice's first insertion and updates its document state. It then sends an acknowledgement to Alice
7. Alice removes that operation from "Sent updates" and updates her latest revision number to 1
8. Server broadcasts Alice's insertion to Bob and Bob applies the operation to his document replica and updates his latest revision number to 1

![47b2fe5b555511a6d467ca9f429ddb0d.png](../_resources/47b2fe5b555511a6d467ca9f429ddb0d.png)

1. Alice's second insertion of " world" can now be processed. The operation is sent to the server and moved from "Local updates" to "Sent updates"
2. The server receives Alice's second request and adds her operation into its "Pending updates" queue
3. At the same time, Bob inserts a "!" character at the end of "Hello"
4. Bob's insertion of "!" is sent to the server and added to the server's "Pending updates" queue. Bob moves the operation from "Local updates" to "Sent updates"
5. Server processes Alice's second insertion first and sends an acknowledgement to Alice
6. Alice removes that operation from "Sent updates" and updates her latest revision number to 2
7. Server broadcasts Alice's insertion to Bob. However, Bob has uncommitted updates, so he needs to transform them against Alice's updates. Bob's insertion of "!" is transformed to the 11th position to make room for Alice's " world". Bob updates his latest revision number to 2
8. Server processes Bob's insertion next. It sees that Bob's operation was made against revision 1, which does not account for Alice's second insertion. Hence the server transforms Bob's insertion of "!" to the 11th position to make room for Alice's " world". This shift is identical to the transformation Bob's client made when it first received Alice's insertion of " world"
9. Server processes Bob's transformed insertion and sends an acknowledgement to Bob. Bob removes that operation from "Sent updates" and updates his latest revision number to 3
10. Server broadcasts Bob's insertion to Alice. It has already been transformed, so Alice can simply apply the operation and update her latest revision number to 3

Alice, Bob, and the server all end up with the same version of the document.

**Note**: Update logs of Alice and Bob aren't shown in the diagrams, but they mirror the server's revision log.

### Transport mechanisms
The collaboration approaches discussed thus far do not restrict or prescribe any transport mechanisms. However, there are certain requirements that the selected transport mechanism needs to fulfill:

- **Bidirectional**: Servers need to be able to initiate sending of data to peer clients whenever an update from a client is received.
- **Low latency**: Although user edits are first made locally and appear instantaneously, and update payloads are small, a low latency transport mechanism will help users receive peer updates and persist changes more frequently, reducing the likelihood of data loss due to crashes.
- **Ordered**: Servers assume that updates broadcasted to clients will be received in the order they are sent as transformations are made with the assumption that the operations are made in order. Hence the transport mechanism needs to guarantee that messages are sent in order during broadcast.

While latency is largely determined by network connection speed and reliability, the protocol also plays a part. Some transport approaches are persistent and therefore more efficient as there is no need for repeated handshakes and initialization. The possible approaches are:

- **Long polling**: In long polling, the client sends a request to the server, and the server holds the connection open until new data is available. Once data is sent, the client immediately reopens the connection to wait for the next update.
- **WebSockets**: WebSockets provide a full-duplex communication channel over a single, long-lived persistent connection. Once a WebSocket connection is established between the client and server, the server can push updates to the client as they occur. Ideal for scenarios where both the client and server need to continuously exchange data, like in chat applications, multiplayer games, or collaborative editing tools.
- **Server-sent events (SSE)**: SSE is a standard that allows the server to push text-based event updates to the client via HTTP. Unlike WebSockets, SSE is unidirectional (server-to-client) and is more suitable for unidirectional scenarios where only the server needs to push data to the client. For client-to-server communication, standard HTTP requests can be used. SSE is well-suited for applications where updates are infrequent or the data is primarily sent from the server to the client.

#### Analysis
**Reliability**: Reliability is the biggest concern here – how do we ensure that messages sent from the server are guaranteed to be received by the client? The connection could be unstable if the user is on the move or at a cafe. Persistent connections could disconnect and require reconnection. When a client is connecting, reconnecting, or offline, it can miss out on messages that happened on the server but could not reach the client.

This issue is especially significant in long polling. While long polling tries to mimic real-time updates, there can be a delay between when new data is available on the server and when it is received by the client. This delay occurs because the server needs to wait for an existing long polling request to return before the client can receive the new data. Reconnection is literally built into how long polling works, so there's a larger window for server updates to be missed.

**Ordering**: None of the mentioned transport mechanisms guarantee in-order delivery of the messages. Client-to-server communication is guaranteed thanks to client-level message ordering by enforcing only one in-flight request at a time. Server-to-client communication ordering is required by the communication protocol but not built into it. In SSE, if the connection drops, it is automatically retried but that's not the case for other methods.

Overall, **WebSockets is the best choice of transport mechanism for collaborative editing given its low-latency** and bidirectional communication properties. WebSockets is widely used by many production-grade apps for real-time updates. Although WebSockets do not guarantee message ordering, that gap can be filled with custom logic in the application layer via libraries like Socket.io.

It is also possible to use SSE. We can use SSE for receiving peer updates and regular HTTP requests for sending messages, but it can be confusing to use two different transport mechanisms for collaborative editing.

Long polling is not viable given there are more modern technologies that overcome its shortcomings.

### Overall approach summary
Let's summarize the key areas of the overall approach.

- **Efficient request payload**: Only the operation kind and necessary information is sent and received during updates, resulting in requests being small and efficient. The payload size isn't affected by the length of the document.
- **Client-server network architecture**: A client-server model is selected where participants connect to a central server. Having a central server simplifies the collaboration protocol easier to understand and implement as clients only have to focus on synchronization with the server and not other clients. The server can treat all clients in the same fashion.
- **Concurrency control via version-detection**: A version-detection model is the most ideal where each client holds a replica of the document and changes are propagated to peers as soon as possible. The document is never locked and every user can optimistically make changes locally without waiting for server acknowledgement. The speed and reliability of the network connection doesn’t limit how fast users can type. Conflicts in versions can be resolved by transforming operations (OT) or handled with special data structures (CRDTs).
- **Conflict resolution via OT**: OT is chosen due to its maturity and OT has its roots in document editing. Each operation provides enough information for merging updates. Google Docs has also stuck with OT technology over the years and is still using OT up till this day.
- **Store documents as a revision log**: The document is stored as an append-only log of operations/changes, which enables version history and efficient computation of differences between client states and server states.
- **Update scheduling**: Each client maintains a "local updates" buffer and "sent updates" buffer, ensuring that there will only be one network request in-flight at any time.
- **Operation granularity**: Operations granularity is decided at the intention level and continuous typing actions are coalesced.
- **WebSockets for transport**: WebSockets are low-latency and bidirectional which is suitable for server-to-client communication. Message out-of-order issues can be addressed via custom logic in the application layer.

We highly recommend checking out this [interactive visualization of OT with a central server](https://operational-transformation.github.io/), it really helps with understanding the collaboration protocol.

Now that we have discussed the various options and decided on an approach, we can reorganize and present them using the RADIO framework sections.

## Architecture / high-level design
The high level architecture has already been covered extensively above. In summary, a client-server architecture is the most suitable given the requirements:

- Central server where all clients communicate with. Clients do not communicate with each other directly.
- Server holds the source of truth as a revision log, which enables computation of the latest document state.
- Clients sync with the server as soon as possible.

![63546c77fbb6f01781bf35dafba7393e.png](../_resources/63546c77fbb6f01781bf35dafba7393e.png)

### Component responsibilities
We'll focus on the key components of a collaborative editor.

- **UI**: Displays the document and sends events to the rich text editor core.
- **Rich text editor core**: Holds the document state/model, similar to the core of the rich text editor system design. Responds to user events by manipulating the underlying document state which triggers DOM events. Provides APIs for external modifications of the document state.
- **Sync engine**: Module responsible for syncing local updates to the server, receiving updates from the server (transforming appropriate operations) and updating the editor core. Most of everything discussed above lives in this module.
	- **Local updates buffer**: Local operations that have not been sent to the server.
	- **Sent updates buffer**: Local operations that have been sent to the server but not yet acknowledged by the server.
	- **Updates log**: Revisions that have been committed as part of the document history.
- **Server**: WebSocket server that can receive and push updates to the client.
	- **Pending updates**: Operations from clients that have not been committed.
	- **Revision log**: Operations that have been processed and can give the complete history of the document.

The editor core and UI is entirely decoupled from the sync engine and server back end. The editor core provides APIs for the sync engine to modify the document model depending on the received operations.

## Data model
The core state to discuss here is the document state. As per the Rich Text Editor system design article, within the rich text editor core, the document state is modeled as a tree.

However within the sync engine, the document is stored as the initial document state + multiple lists (sequences) of operations (local, sent, received). The latest document state can be constructed by applying these operations on top of the initial document state. Each committed operation increases the revision number.

![3654cb2e686683d74a43e0c16add847e.png](../_resources/3654cb2e686683d74a43e0c16add847e.png)

When a new user joins the session or if the current user refreshes the page, the document they initialize with is the latest version, constructed from all the committed operations on the server; all client-side operations lists will be empty.

![56e3e0fa14636caf18c410fad8880efa.png](../_resources/56e3e0fa14636caf18c410fad8880efa.png)

The diagram above demonstrates the states of clients who join at different times. Alice joins the session when the document is at revision 1 and only contains "Hello". She adds " world" to the document and sees "Hello world" on her screen. Alice's document was constructed from an initial document state of "Hello" and her insertion operation of " world".

Bob then joins the session at revision 2 and he is directly initialized with the current document state of "Hello world", without any granular operations history.

### Operational transformations on the document level
So far the operations (and transforms) we have mentioned in this article are executed on the sentence level in a plaintext context. However, documents are rich text, how can transformations be performed on documents?

In the Rich Text Editor system design article, we have discussed how tree data structures can be used to represent rich text documents, hence we need OTs that can be used on tree structures. Discussing the technicalities of OTs is beyond the scope of this article, but we will briefly discuss the rough approach.

A full rich text document is represented as a tree and there are two categories of nodes – element nodes and text nodes. Element nodes can contain child nodes, which are other element nodes or text nodes, while text nodes are leaves and can only contain textual content (plaintext) and have formatting flags. Heading and paragraphs are subclasses of element nodes because they can contain child text nodes. At a high level, a document contains a root node with a list of heading elements, paragraph elements, as its direct children etc.

Operational transformations work well on strings, which are list-like structures. The children of element nodes are also in a list structure, see the similarity here?

Let's look at some potential scenarios when editing a document:

- **Inserting characters in the same sentence**: Let's assume the sentence is contained within a single text node. We've covered such conflicts pretty thoroughly above. The insertion at the back will have to be transformed (offset increased) to make space for the insertion in front.
- **Inserting paragraphs at the same time**: Let's assume these paragraphs are direct children of the document root node. Inserting paragraphs is equivalent to modifying direct children of the root node, a list-like structure. Notice that inserting characters within the same sentence is also modifying a list (of characters) where the insertion operation at the back needs to be shifted. Hence a similar transformation can be used to resolve the conflict for paragraph insertions.
- **Inserting characters in different paragraphs**: There's no conflict here since different nodes are modified.

The pattern here is that conflicts will arise when the same nodes are modified, which thankfully isn't that common in longer documents. OT techniques can be used to resolve conflicts in lists, whether they are a list of characters (in sentences and paragraphs) or a list of paragraphs (in the document's root node).

## Interface definition (API)
We'll focus on the core APIs required for syncing between client and server.

### Initialization API
This API provides the client with the necessary information to start a collaborative editing session. A sample response looks like this:

```js
{
  "revision": 145,
  "document": "..." // Rich text editor format
}
```

### Update/save API
This API allows sending of local updates to the server. Since this is done using WebSockets, a type field is necessary to differentiate between the requests. Note that multiple operations are allowed in a single update request.

#### Request

```js
{
  "type": "UPDATE",
  "requestId": 2, // Monotonically increasing integer per client
  "revision": 146, // Base revision that the update is performed on
  "isUndo": false, // Differentiate between new and undo operations
  "operations": [
    {
      "type": "INSERT",
      "nodeId": 24, // Needed in a document context
      "payload": {
        "characters": "Hello",
        "index": 4
      }
    },
    {
      "type": "INSERT",
      "nodeId": 25,
      "payload": {
        "characters": "Bye",
        "index": 2
      }
    }
  ]
}
```

### Update acknowledgement callback
The server sends this to the client upon acknowledgement of an update request. Upon acknowledgement, clients can move the operation from the "Sent updates" to the "Updates log".

```js
{
  "type": "ACK",
  "requestIdAcknowledged": 2,
  "requestId": 3,
  "revision": 147
}
```

### On update callback
These are server-initiated messages that indicate a peer made an update.

```js
{
  "type": "PEER_UPDATE",
  "revision": 148,
  "userId": 6543, // User who made the update
  "operations": [
    {
      "type": "INSERT",
      "nodeId": 24, // Needed in a document context to identify the node to modify
      "payload": {
        "characters": "Goodbye",
        "index": 8
      }
    },
    {
      "type": "INSERT",
      "nodeId": 25,
      "payload": {
        "characters": " earth",
        "index": 4
      }
    }
  ]
}
```

### On reconnect callback
When clients disconnect and finally reconnect, the server should send it any revisions it has missed out on while the client was disconnected.

```js
{
  "type": "SYNC",
  "revisions": [
    {
      "revision": 147,
      "userId": 6543, // User who made the update
      "operations": [
        {
          "type": "INSERT",
          "nodeId": 24, // Needed in a document context to identify the node to modify
          "payload": {
            "characters": "Goodbye",
            "index": 8
          }
        },
        {
          "type": "INSERT",
          "nodeId": 25,
          "payload": {
            "characters": " earth",
            "index": 4
          }
        }
      ]
    }
  ]
}
```

## Optimizations and deep dive
### History and versioning
Google Docs allow users to view the document history as a list of versions. By storing the document as a log of operations/updates, we can "time travel" and go back to the document state at any point in time. Each document revision is identified by a monotonically increasing positive integer, which is constructed from the operations up till that point.

![0cd27aa955156d65b2101f818e5be21f.png](../_resources/0cd27aa955156d65b2101f818e5be21f.png)

Although every granular change is stored in the database, it is more meaningful to display a version log, which groups multiple revisions together. Changes made together within a short duration are part of the same version:

![4fa8e60c97d8e6bd4032cae07110ca4d.png](../_resources/4fa8e60c97d8e6bd4032cae07110ca4d.png)

This is similar to what Google Docs displays when you click on the "Version History" button.

### Undo/redo
Undo/redo is a tricky topic for rich text editors and even more so when it is a collaborative one.

Should a user's undo/redo history be on the user level or document level (all participants share the same undo/redo)? It makes more sense to only undo your own actions as users are unlikely to be aware of what others are doing and want to undo them.

The document's revision log is append-only; we can only add, not remove. At the same time, removing already committed operations is complex as that might require other clients to undo certain transforms. Google Docs implements undo by appending the negation of the previous operation as a new update. Clients can filter their "Updates log" for changes that were made by them and append negations of those operations.

Hence update operations need an isUndo flag to differentiate between new operations vs undo operations and filter them out when identifying their last non-undo operation, otherwise users will be stuck at undo-ing/redo-ing (undo-ing the undo) the most recent operation.

![20ae38f349ce94c9b3f6778917ff43fd.png](../_resources/20ae38f349ce94c9b3f6778917ff43fd.png)

The revision log above demonstrates how Erin's insertions are undone as deletion operations (a negation of the insertion) appended to the revision log as revisions 7 and 8.

### Reliability
Reliability is about ensuring that messages sent from the server are guaranteed to be received by the client, in the order they were sent. Clients can disconnect anytime, resulting in the possibility of the server sending a message and not being received by the client. Clients can also be on unstable network connections and some messages get lost in transition.

Dropped messages are not an issue if the messages contain the entire document state, but that as explained above, it isn't efficient. In our approach, messages contain crucial granular updates and missing out on any of them will result in document replicas going out-of-sync. Clients have to receive all peer operations in order to converge to the consistent document state.

This is where document revision numbers come in useful, if the server is aware of the document revision the client has, it can use that information to compute what updates are missing on the client.

Who should keep track of each client's latest document revision number? It is troublesome and also not scalable for servers to keep track of the document revision number of each client. A scalable approach involves clients maintaining the latest document revision number they have and including that value in server requests. That way, the server can remain relatively stateless.

If a client reconnects to the session with revision 5 in the payload and the server is currently on revision 8, the server knows that the client is missing revisions 6, 7, and 8, and thus can send the updates of revisions 6, 7, and 8.

Other reliability requirements include in-order delivery, retries, and acknowledgements. WebSockets do not include these features, so custom logic has to be added into the application layer via libraries like Socket.io.

The collaboration protocol outlined above can handle flaky connections well. As long as the client retains the document revision number and sends it to the server as part of the request payload, the server will be able to determine what updates are missing from the client. Each update operation also has a unique ID tagged to them, which facilitates de-duplication in the case of duplicate requests.

### Offline editing
As of writing (Aug 2024), Google Docs does not support offline editing. However, offline editing can be supported relatively effortlessly with the current architecture. When the client detects that there is no network connection, users can continue editing, but any updates remain in the "local updates" buffer and are not sent out.

When the client regains network connectivity:

- **Send local updates to server**: The "local updates" are sent to the server and moved to the "send updates" buffer.
- **Fetch updates from server**: The client could have missed out on some updates while it was offline, the server should push the missing updates since the client's last synced revision.

### Document formats
There are numerous file formats that are supported by word processors like Google Docs and Microsoft Word. Out of which, the .docx (Microsoft Word) and .odt (OpenDocument Text) file formats are the most popular.

The specification for the .docx and .odt file formats are openly available:

- `.docx`
- `.odt`

While Google Docs can open these file formats, it doesn't mean the internal document state matches them exactly. As long as the word processor includes modules to import and export between external formats and its internal state, the software is free to use any format internally.

Let's briefly look at what goes into `.odt `files. `.odt` files are OpenDocument Text files, a format used primarily by word processing applications like LibreOffice Writer and Apache OpenOffice. They are similar to `.docx` files used by Microsoft Word but are based on the OpenDocument format, an open standard for document file types.

An `.odt` file is essentially a ZIP archive that contains several XML files and directories, each serving a specific purpose in storing the content, styles, settings, and other aspects of the document. Here are examples of the main XML files within an `.odt` file:

`content.xml`: This is the core file that contains the actual text content of the document along with its structure. It includes elements like paragraphs, tables, lists, and other document elements in XML format.

```xml
<office:text>
  <text:p>This is a paragraph of text in the document.</text:p>
  <text:table>
    <!-- Table data here -->
  </text:table>
</office:text>
```

`styles.xml`: This file defines the styles used in the document, such as paragraph styles, character styles, table styles, and page layouts. It ensures consistent formatting across the document.

```xml
<office:styles>
  <style:style style:name="Heading1" style:family="paragraph">
    <style:text-properties fo:font-size="18pt" fo:font-weight="bold"/>
  </style:style>
</office:styles>

```

`meta.xml`: Contains metadata about the document, such as the title, author, creation and modification dates, word count, and other descriptive information.

```xml
<office:meta>
  <meta:initial-creator>John Doe</meta:initial-creator>
  <dc:title>My Document</dc:title>
  <meta:creation-date>2024-08-14T10:00:00</meta:creation-date>
</office:meta>
```

`settings.xml`: Stores various settings related to the document, such as page view options, printer settings, and other user preferences that affect how the document is displayed or printed.

```xml
<office:settings>
  <config:config-item-set config:name="ooow:ViewSettings">
    <config:config-item config:name="ViewMode" config:type="string">Normal</config:config-item>
  </config:config-item-set>
</office:settings>
```

`manifest.xml`: This file lists all the files contained within the `.odt` archive and their MIME types. It serves as a directory for the contents of the archive.

```xml
<manifest:manifest>
  <manifest:file-entry manifest:full-path="/" manifest:media-type="application/vnd.oasis.opendocument.text"/>
  <manifest:file-entry manifest:full-path="content.xml" manifest:media-type="text/xml"/>
</manifest:manifest>
```

#### Other files:

- **Images**: If the document contains images, they are stored as separate files within the package.
- **Embedded objects**: Other types of embedded objects, such as charts or spreadsheets, might also be included as separate files.

These XML files work together to define the content, appearance, metadata, and settings of the document, allowing it to be opened, edited, and displayed consistently across different word processing software that supports the OpenDocument format.

## References
- [Concurrency Control in Groupware Systems](https://dl.acm.org/doi/pdf/10.1145/67544.66963)
- [High-Latency, Low-Bandwidth Windowing in the Jupiter Collaboration System](https://dl.acm.org/doi/pdf/10.1145/215585.215706)

















