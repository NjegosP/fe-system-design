# Chat App (e.g. Messenger)

## Question
Design a chat application that allows users to send messages to each other.

## Requirements exploration
What are the core functionalities needed?
- Sending a message to a user.
- Receiving messages from a user.
- See one's chat history with a user.

### Is the message receiving real-time?
Yes, users should receive messages in real-time, as fast as possible without having to refresh the page.

### What kind of message formats should be supported?
Let's support formats text which can contain emojis. We can discuss supporting images if there's time.

### Does the application need to work offline?
Yes, where possible. Outgoing messages should be stored and sent out when the application goes online and users should still be allowed to browse messages even if they are offline.

### Are there group conversations?
We can assume it's a 1:1 messaging service.

## Architecture / high-level design
The main difference between traditional applications vs chat applications that can be used offline is that if the application loses network connectivity, some functionality like browsing on-device messages and searching should ideally still work. This greatly influences the application architecture such and it will differ a great deal from conventional web applications.

### Tricky scenarios to be Handled
Firstly, let's be aware of the various tricky scenarios that we need to handle in a chat application and their implications.

- **Using the application on different tabs in the same browser**. The user might do this because they want to chat with different people at once and rather switch between tabs than switch between conversations in the same tab.
	- **Users should see the same messages for each conversation** -> Rely on a storage that is accessible across different tabs in the same browser.
- **Using the application on different devices/browser.** The same device, different browser case is rare, but it's not rare for users to be using multiple devices at once (work and personal device at the same time).
	- **User should see the same messages for each conversation** -> Sync with the server on initial load and get the most updated data.
- Application goes offline during usage. User could have lost connectivity while on the move and passing through low connectivity areas (common occurrence in the subway).
	- **Outgoing messages have not been all fulfilled** -> They should be retried when the application goes online again, or their statuses should be updated if they were written to the server but didn't receive the updates.
	- **Messages are being sent while application is offline** -> These messages should be sent out when the application next goes online. However, this should only be done for messages sent recently. If these messages were sent too long ago, the conversation might have already progressed past the topic (possibly using other devices) and it no longer makes sense to retry sending it.
- **A combination of the above scenarios.** Life just became a lot harder!
Our chosen architecture should handle all these scenarios.

### Client-side database
One way to store data on the client is to use a client-side database (henceforth referred to as database). The UI reads data from the database as if it's a client-only application as opposed to traditional applications where the UI directly makes HTTP queries and displays the fetched data. The UI is not and should not be aware of where the database got its data from. Where the database gets its data from should be an implementation detail of the data layer.

Different tabs in the same browser access the same client-side database. This ensures consistent data between tabs and helps to solve the UI consistency issue in the "using the application on different tabs in the same browser" scenario. However, we have to take note of not inserting into the database twice when we are notified of a "new message" event.

### Data syncer
The Data Syncer is a module that is in-charge of syncing the client database with the server.

#### Sending messages
When the user sends out a message (or any update made by the user in general), we want to reflect the changes immediately. It is poor user experience to wait for the server's confirmation before showing an updated UI.

Hence outgoing chat messages/user actions are first inserted into the database instead and they are marked as pending. Pending messages are also reflected immediately in the UI. Notice that messages in chat applications have indicators to indicate the various message delivery statuses.

![da529e1d3befcaa5d53ddacd1c7e648a.png](../_resources/da529e1d3befcaa5d53ddacd1c7e648a.png)

You might have heard of the phrase "she double blue ticked me" to mean someone read the messages without replying. Now you know what the other message statuses are 😎.

During database syncing, after the server has received and acknowledged the action, it sends back a response to the application and those messages can be marked as "Sent".

As there can be multiple messages being sent in parallel across different conversations (and in real applications, even more actions like reactions, deleting messages), there is a need for a scheduler to ensure that actions are sent to the server in the right order, request statuses are tracked, request failures are retried, etc.

#### Receiving real-time updates
Because we want to receive message updates in real-time, the application needs to be informed about new messages from the back end as soon as possible. We'll discuss a few ways of getting real-time updates in the "Optimizations" section.

### Server-side rendering or Client-side rendering?
Chat applications have the following characteristics:

- Highly interactive in nature due to high frequency of sending and receiving messages. The page will probably need a fair amount of JavaScript.
- Messages can only be accessed when logged in.
- Messages do not have to (and should not!) be indexed by search engines.
- Fast initial loading speed is desired but not the most crucial.

Considering the above, a pure client-side rendering and single-page application overall architecture will work well. We can use server-side rendering (SSR) with client-side hydration like in News Feed system design and Photo Sharing application system design for a fast initial load but the benefits of SSR will only be limited to performance gains because chat applications don't need to be SEO-friendly. The additional engineering complexity of enabling SSR might not be worth it.

### Architecture diagram

![7dad5602de8909ef2cb5909aa1d92833.png](../_resources/7dad5602de8909ef2cb5909aa1d92833.png)

### Component responsibilities
- **Chat UI**: Contains a list of conversations and the currently selected conversation/conversation
	- **Conversations List**: Presents a list of conversations (user, last message, last message timestamp).
	- **Selected Conversation**: A list of messages in the conversation and a input box to type new messages.
- **Controller**: Controls the flow of data within the application. Fetches data from the database to show in the UI. Writes data to the database.
- **Data Syncer**: Module that contains the database and also manages the outgoing messages. Also receives updates from the server and updates the database accordingly.
	- **Client-side Database**: Database to store all the data needed to be shown in the UI.
	- **Message Scheduler**: Monitors the outgoing messages, schedules them for sending and manages their statuses.
- **Server**: Provides HTTP APIs and real-time APIs to fetch conversation and message data and to send new messages.

## Data model
To keep it simple, we will focus only on the chatting functionality of the application.

### Client-side database
Most of the data needed by the application will be stored in the client-side database. Any data that is needed for offline functionality should go into the database. Here's an entity-relationship diagram of the database tables.

![b32a8faa5cd9e6da39cc760a61cfdbcd.png](../_resources/b32a8faa5cd9e6da39cc760a61cfdbcd.png)

![f7fb1b64f628b9c13343b1ec0eb0a1d9.png](../_resources/f7fb1b64f628b9c13343b1ec0eb0a1d9.png)

Note that the DraftMessage and SendMessageRequest tables are not synced to the server and are client-side only. However, they should still be in the database rather than be client-only state as they should be persisted across sessions.

- `DraftMessage`: This table stores messages the user has typed in a conversation's message input box but hasn't sent. This has to be persisted into the database (as opposed to client-only state) so that if the user quits the application and opens it again, they don't lose their unsent message. Each conversation can have a maximum of one `DraftMessage` per user.
	- Note that draft messages aren't synced with the server, so it stays within the current device. It's totally doable to sync draft messages with the server so that they are accessible across all devices, but that's a product decision and we won't explore this now for the sake of focusing on the core use cases.
- `SendMessageRequest`: This table stores data related to each message that the user has sent but hasn't been acknowledged by the server. `status` is an enum and can be one of the following:
	- `pending`: The default state of new messages to be sent.
	- `in_flight`: The application has sent the message to the server but hasn't received a response.
	- `fail`: When the server returns an error or the send request has timed out. We keep track of the number of times it has failed in `fail_count` so that we know whether to keep retrying (with exponential backoff) or stop retrying after a certain number of failures.
	- `success`: Indicates that the message has been received and acknowledged by the server. Strictly speaking, this enum value is not needed because when the client receives the server acknowledgement, this row can be removed from the table.

### Client-only state
These are state fields that do not need to be persisted in the database, i.e. it's ok to lose this data if the user exits the application by closing the browser tab/window.

- **Selected conversation**: The currently selected conversation.
- **Conversation scroll position**: Scroll position within each conversation. It's useful to restore the scroll position whenever the user switches between conversations.
- **Conversation outgoing message**: This is whatever the user is typing in a specific conversation. It is almost the same as the `DraftMessage` except we shouldn't save into the database on every keystroke. We only persist to the `DraftMessage` table after the user has stopped typing (via blur/debounce) or throttle to save the value to the database after every X ms.

## Interface definition (API)
The following APIs are needed:

- Send message
- Sync outgoing messages
- Server events
- Fetch conversations
- Fetch conversation messages

### Send message
1. Add a row to the `Message` table with a `sending` status.
2. Add a row the `SendMessageRequest` table with `pending` status.
3. Conversation UI reads from the `Message` table and shows the new message with a "Sending" indicator.
4. Any `DraftMessage` rows for the current conversation/conversation are deleted.
5. At this point, there are no synchronous steps left to be done. The Message Scheduler will be in-charge of syncing the `pending` messages with the server.

### Sync outgoing messages
The Message Scheduler will be in-charge of syncing outgoing messages with the server. It will maintain its own task queue and watch the `SendMessageRequest` table. Because the task queue needs to be in-sync across tabs, it should not be stored in browser memory and can also use another table.

Whenever the table is not empty, it will fetch the first X rows (ordered by id) and try to process them by adding tasks to its own task queue. X is a configurable value. Depending on the row's status column:

- `pending`: Enqueue a task to send the message to the server via the real-time channel. Update the row's `status` to be `in_flight`.
- `in_flight`: Check the last_sent_at timestamp. If it has exceeded the timeout threshold, update the row's `status` to be `fail` and update the `fail_count` by 1.
- `fail`: Enqueue a task to retry sending this message sometime in the future. The delay duration depends on the `fail_count`. With an exponential backoff retry strategy, the delay duration will increase exponentially with the `fail_count`.

### Server events
The Data Syncer will receive real-time updates from the server in the form of events. Each event can have a type and a payload field. The shape of the payload depends on the actual event.

```js
// Example event payload pushed from the server.
{
  "event_name": "incoming_message",
  "payload": {
    "foo": "value_a",
    "bar": "value_b"
  }
}
```

These are the various events necessary:

### `message_sent` event
- Update the `Message`'s `status` to be `sent`.
- Clean up this message in the Message Scheduler:
	- Remove the row corresponding to this message in the `SendMessageRequest` table. This message has been received by the server and is no longer `pending` or `in_flight`.
	- Remove any tasks in the task queue that are related to this message.
- Update UI
- Notify the Conversation UI to be updated if that message's conversation is currently shown.

### `message_delivered` event
- Update the `Message`'s `status` to be `delivered`.
- Update UI
- Notify the Conversation UI to be updated if that message's conversation is currently shown.

### `message_failed` event
- Update the row corresponding to this message in the `SendMessageRequest` table's and change the `status` to be `fail` and `increment` `fail_count` by 1.
	- Note that we don't modify the `status` of the row in the `Message` table to be `fail` yet. The message is not deemed to be failed yet until we have retried sending it.
- Update UI
- Notify the Conversation UI to be updated if that message's conversation is currently shown.

### `incoming_message` event
- Append the new message into the `Message` table.
	- Create a new row in the `Conversation` table if it doesn't exist.
	- Create a new row for the message's sender into the `User` table if it doesn't already exist.
- Update UI
	- Notify the Conversations List UI to be updated. Update the UI to surface this conversation to the top. If the Conversations List is sorted by decreasing timestamp of each conversation's latest message, it will be automatically surfaced to the top.
	- Notify the Conversation UI to be updated if that message's conversation is currently shown.

### `sync` event
**Work in progress.** This event is triggered when a client first connects to the server. When clients first connect to the server, they might be lagging behind in terms of the data it contains The goal here is to get each client up-to-date with the latest server state by the server sending down all data that is missing on the client.

Possible ways of indicating the client state to the server:

- **Client's last update timestamp**: The server will gather all the new entities (messages, conversations) created after the timestamp and send to the client for the client to insert into the db.
- **Each conversation's cursor**: A database cursor is a mechanism that enables traversal over records in a database. Similar to the cursor-based pagination APIs, a cursor can be used to indicate the last message in the conversation that the client received, and what messages are after the message.

## Optimizations and deep dive
### Client database
#### Deciding on client-side storage

There are a few ways of storing data on the client: Cookies, Web Storage, and IndexedDB. Refer to the quiz question for a [comparison between cookies and Web Storage mechanisms](https://www.greatfrontend.com/questions/quiz/describe-the-difference-between-a-cookie-sessionstorage-and-localstorage).

Cookies is out of the question due to the extremely small capacity (4kb per domain). `localStorage` (one of the Web Storages) is not well-suited because it doesn't support structured data, which is essential for non-trivial applications like chat.

[IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) is our best choice here, which is a low-level API for client-side storage of significant amounts of **structured data**, including files/blobs. Other useful features include database indexes, tables, cursors, transactions, mostly over asynchronous APIs.

### Syncing across tabs
Since IndexedDB is a client-side storage mechanism, the data is accessible across tabs and it solves the "users should see the same messages in the application on different tabs in the same browser." scenario outlined at the top.

However, browsers tabs are not aware of `IndexedDB` data changes in other tabs. To inform the other tabs about changes in the database, use the `BroadcastChannel` which allows communication between different windows/tabs/frames of the same origin.

### Syncing client database with the server
Bidirectional syncing of messages between the client and server is complex.

- **Out-of-order messages**: There is no guarantee that messages are received in the order they are sent. Should out-of-order messages be inserted in between existing messages according to timestamp or should they always be appended at the bottom of the conversation?
- **Fetching new messages**: The client needs to tell the server when the last update was (could be the last message received, or the timestamp it last pulled from the server) and the server figures out what messages have not been sent to the client and send them.
- **Sending out pending messages**: Messages sent while the application is offline should be stored in a pending outgoing messages queue and sent out when the application goes online.

### Other issues
- Unsupported environments like private/incognito mode on Firefox and Safari.
- Storage limits.
- Errors initializing/opening the database.
- [Best Practices for Using IndexedDB](https://web.dev/articles/indexeddb-best-practices).
- IndexedDB comes with a [number of problems, bugs and oddities](https://gist.github.com/pesterhazy/4de96193af89a6dd5ce682ce2adff49a)

### Real-time updates
Real-time messaging means that recipients of the messages receive new messages instantly (or near instantly) without them having to reboot the application/refresh the page or manually trigger a button to fetch new messages.

The few ways of implementing real-time messaging:

- Short Polling (or Regular Polling)
- Long Polling
- Web Sockets

**In Progress**: Evaluate the pros and cons of each real-time mechanism. Web Sockets is the modern choice and the mechanism used by most chat apps.

Reference: [WebSockets vs Long Polling](https://ably.com/blog/websockets-vs-long-polling)

### Network
Connection failure is extremely common as users could be using chat applications while on transport and going in and out of poor connectivity areas. Messages could fail to send out, along with other issues:

- **Offline Usage**: The application should detect if the device is offline and not attempt to send messages if so. The messages should be added to the `SendMessageRequests` table.
- **Failures**: Failed outgoing messages should be retried using exponential backoff.
	- Display an error message if after X number of retries the messages weren't successfully delivered.
- **Batching**: Outgoing messages could be batched and sent as a single message if messages are sent in quick succession (within a few seconds). The application can detect if the user is still typing after their last sent message and perhaps wait for the next message to be completed before making the request (similar to debouncing). This batching logic is best implemented within the Message Scheduler.
- **Out-of-order**: If we send each messages in a separate request, there is no guarantee that the messages reaches the server in the order they were being sent from the client. However, sequentially sending messages is also less than ideal. Batching helps to mitigate this problem by sending multiple messages in one payload but also maintaining the order.
- **Disconnection**: App should automatically attempt reconnection after a disconnection without the user having to refresh the page.

### Performance
- Lazy load components that aren't needed for initial load (e.g. emoji picker, any popovers/modals).
- Leverage windowing/virtualization for long list of messages within a conversation.

### Accessibility
Implement basic keyboard shortcuts:

- Message composer
	- Enter to send a message.
	- `Shift` + `Enter` to add new lines within a message.
- Between conversations
	- Shortcut to focus on the search bar.
	- `Ctrl`/`Cmd` + `Up`/`Down` to traverse between conversations.
	- On certain desktop clients, `Ctrl`/`Cmd` + `number` brings you to the nth conversation in your list of conversation.

### Offline support
The application can be built as a Progressive Web Application (PWA) which uses service workers to cache assets (HTML/JS/CSS) so that the application has both the code and data required to be used offline.

Using PWAs also allow for browser notifications which is useful for informing the user that there's a new message even when the tab is not in focus/visible.

### User experience
#### Maintaining scroll position
Scroll position is a tricky issue to deal with in messaging apps due to the possibility of new messages being added to the list above and below.

Scroll position should be:

1. Kept to the bottom of the message list when new messages come in and the scroll position is already at the bottom of the list. This is the default scenario for most situations.
2. Maintained when the scroll position is not at the bottom such that the currently visible contents are still visible. Scrolling up to read the older chat messages scroll position should be maintained and current visible elements should not move even though more DOM elements will be added to the top. The application can calculate the current scroll offset, the height of the new elements to be appended, and modify the scroll height to add in the new elements height.

The following are events that can change potentially change the (scroll/client) height:

- New messages inserted below (when receiving new messages).
- New messages inserted above (when searching for history).
- Window resizing.
- Media loading completely that has a different height from the loading placeholder.
	- Avoid this issue by using a fixed height placeholder and rendering the media within that element.
- Page zoom changes.
The scroll positions should be maintained (either at the bottom or showing the same contents) depending on the scenarios listed above.

### Other possible improvements
- Add a "Scroll to Bottom" button that's visible when the user has scrolled upwards within the conversation messages.

### Gradient effect
[This article by CSS Trick](https://css-tricks.com/recreating-the-facebook-messenger-gradient-effect-with-css/) shows you various approaches for implementing Messenger's chat message gradient background.

### Stale clients
For extremely stale clients, they will have to download the entire list of absent messages since the last sync which can be huge. This leads to a significant delay between starting up the application and being able to use it. Few will enjoy the process of waiting for tens of thousands of messages to be fetched and inserted into the client database before they can use the app.

A plausible approach is to treat it as a fresh load/existing data in the database as non-existent and do a complete sync with the server, only fetching the latest N messages of the latest M conversations.

### Advanced
These features won't be discussed in this solution but if time permits you might want to discuss them with your interviewer.

- Searching (Use a hybrid of both online and offline search)
- i18n
- End-to-end encryption
	- [Challenges of E2E Encryption in Facebook Messenger](https://www.youtube.com/watch?v=-IXJ7Q01gpY)
- Delivery/read receipts
- Offline/optimistic reads
- Reactions
- Typing indicator
- Disappearing messages
- Notifications

## References
- Facebook & Messenger
	- [Launching Instagram Messaging on desktop](https://engineering.fb.com/2022/07/26/web/launching-instagram-messaging-on-desktop/)
	- [Reverse engineering the Facebook Messenger API](https://intuitiveexplanations.com/tech/messenger)
	- [Facebook Messenger Engineering with Mohsen Agsen](https://softwareengineeringdaily.com/2020/03/31/facebook-messenger-engineering-with-mohsen-agsen/)
	- [F8 2019: Facebook: Lighter, Faster, Simpler Messenger](https://www.youtube.com/watch?v=ulVLD2yzCrc)
	- [Building Real Time Infrastructure at Facebook - Facebook - SRECon2017](https://www.youtube.com/watch?v=ODkEWsO5I30)
	- [Facebook Messenger RTC – The Challenges and Opportunities of Scale](https://www.youtube.com/watch?v=F7UWvflUZoc)
	- [Building Mobile-First Infrastructure for Messenger](https://engineering.fb.com/2014/10/09/production-engineering/building-mobile-first-infrastructure-for-messenger/)
	- [MySQL for Message - @Scale 2014 - Data](https://www.youtube.com/watch?v=eADBCKKf8PA)
	- [Project LightSpeed: Rewriting the Messenger codebase for a faster, smaller, and simpler messaging app](https://engineering.fb.com/2020/03/02/data-infrastructure/messenger/)
- Slack
	- [Managing Focus Transitions in Slack](https://slack.engineering/managing-focus-transitions-in-slack/)
	- [Gantry: Slack’s Fast-booting Frontend Framework](https://slack.engineering/gantry-slacks-fast-booting-frontend-framework/)
	- [Making Slack Faster By Being Lazy](https://slack.engineering/making-slack-faster-by-being-lazy/)
	- [Making Slack Faster By Being Lazy: Part 2](https://slack.engineering/making-slack-faster-by-being-lazy-part-2/)
	- [Getting to Slack faster with incremental boot](https://slack.engineering/getting-to-slack-faster-with-incremental-boot/)
	- [Service Workers at Slack: Our Quest for Faster Boot Times and Offline Support](https://slack.engineering/service-workers-at-slack-our-quest-for-faster-boot-times-and-offline-support/)
	- [Localizing Slack](https://slack.engineering/localizing-slack/)
- Airbnb
	- [Messaging Sync — Scaling Mobile Messaging at Airbnb](https://medium.com/airbnb-engineering/messaging-sync-scaling-mobile-messaging-at-airbnb-659142036f06)