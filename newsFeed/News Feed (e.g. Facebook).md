# News Feed (e.g. Facebook)

Designing a news feed application is a classic system design question, but almost no existing resource discusses in detail about how to design the front end of a news feed.

## Question
Design a news feed application that contains a list of feed posts users can interact with.

### Requirements exploration
#### What are the core features to be supported?
- Browse news feed containing posts by the user and their friends.
- Liking and reacting to feed posts.
- Creating and publishing new posts.

Commenting and sharing will be discussed further down below but is not included in the core scope.

#### What kind of posts are supported?
Primarily text and image-based posts. If time permits we can discuss more types of posts.

#### What pagination UX should be used for the feed?
Infinite scrolling, meaning more posts will be added when the user reaches the end of their feed.

#### Will the application be used on mobile devices?
Not a priority, but a good mobile experience would be nice.

## Architecture / high-level design
![eefab33b48111597098f43136526d8d5.png](../_resources/eefab33b48111597098f43136526d8d5.png)
### Component responsibilities
- `Server`: Provides HTTP APIs to fetch feed posts and to create new feed posts.
- `Controller`: Controls the flow of data within the application and makes network requests to the server.
- `Client store`: Stores data needed across the whole application. In the context of a news feed, most of the data in the store will be server-originated data needed by the feed UI.
- `Feed UI`: Contains a list of feed posts and the UI for composing new posts.
   - `Feed posts`: Presents the data for a feed post and contains buttons to interact with the post (like/react/share).
   - `Post composer`: WYSIWYG (what you see is what you get) editor for users to create new feed posts.

### Rendering approach
Traditional web applications have multiple choices on where to render the content, whether to render on the server or the client.

- **Server-side rendering (SSR)**: Rendering the HTML on the server side, which is the most traditional way. Best for static content that require SEO and does not require heavy user interaction. Websites like blogs, documentation sites, e-commerce websites are built using SSR.
- **Client-side rendering (CSR)**: Rendering in the browser, by dynamically adding DOM elements into the page using JavaScript. Best for interactive content. Applications like dashboards, chat apps are built using CSR.
Interestingly, news feed applications are somewhere in-between, there's a good amount of static content but they also require interaction. In reality, Facebook uses a hybrid approach which gives the best of both worlds: a fast initial load with SSR then hydrating the page to attach event listeners for user interactions. Subsequent content (e.g. more posts added once the user reaches the end of their feed) and page navigation will use CSR.

Modern UI JavaScript frameworks like React and Vue, along with meta frameworks like Next.js and Nuxt support this rendering strategy.

Read more on [Rendering on the web](https://web.dev/articles/rendering-on-the-web) and ["Rebuilding our tech stack for the new Facebook.com"](https://engineering.fb.com/2020/05/08/web/facebook-redesign/) blog post.

## Data model
A news feed shows a list of posts fetched from the server, hence most of the data involved in this application will be server-originated data. The only client-side data needed is form state for input fields in the post composer.

![eb3372b60b9b4a21a7e7b915d4366709.png](../_resources/eb3372b60b9b4a21a7e7b915d4366709.png)
Although the `Post` and `Feed` entities belong to the feed post and feed UI respectively, all server-originated data can be stored in the client store and queried by the components which need them.

The shape of the client store is not particularly important here, as long as it is in a format that can be easily retrieved from the components. New posts fetched from the second page should be joined with the previous posts into a single list with the pagination parameters (`cursor`) updated.

### Advanced: Normalized store
Both Facebook and Twitter use a normalized client side store. If the term "normalized" is new to you, have a read of [Redux's documentation on normalizing state shape](https://redux.js.org/usage/structuring-reducers/normalizing-state-shape). In a nutshell, normalized data stores:

- Resemble databases where each type of data is stored in its own table.
- Each item has a unique ID.
- References across data types use IDs (like a foreign key) instead of having nested objects.

Facebook uses Relay (which can normalize the data by virtue of knowing the GraphQL schema) while Twitter uses Redux as seen from the "[Dissecting Twitter's Redux Store](https://medium.com/statuscode/dissecting-twitters-redux-store-d7280b62c6b1)" blog post.

The benefits of having a normalized store are:

- **Reduced duplicated data**: Single source of truth for the same piece of data that could be presented in multiple instances on the UI. E.g. if many posts are by the same author, we're storing duplicated data for the `author` field in the client store.
- **Easily update all data for the same entity**: In the scenario that the feed post contains many posts authored by the user and that user changes their name, it'd be good to be able to immediately reflect the updated author name in the UI. This will be easier to do with a normalized store than a store that just stores the server response verbatim.

In the context of an interview, we don't really need to use a normalized store for a news feed because:

- With the exception of the user/author fields, there isn't much duplicated data.
- News feed is mostly for consuming information, there aren't many use cases to update data. Feed user interactions such as liking only affect data within a feed post.
Hence the upsides of using a normalized store is limited. In reality, Facebook and Twitter websites contain many other features that will benefit from the features a normalized store provides.

Further reading: [Making Instagram.com faster: Part 3 — cache first](https://instagram-engineering.com/making-instagram-com-faster-part-3-cache-first-6f3f130b9669)

## Interface definition (API)
![7affedb8d6275940fd40530d2c301131.png](../_resources/7affedb8d6275940fd40530d2c301131.png)
The most interesting API to talk about will be the HTTP API to fetch a list of feed posts because the pagination approach is worth discussing. The HTTP API for fetching feed posts from the server has the basic details:

![f964040335bfd3ba2e8474f49f48454f.png](../_resources/f964040335bfd3ba2e8474f49f48454f.png)

There are two common ways to return paginated content, each with its own pros and cons.

- Offset-based pagination
- Cursor-based pagination

### Offset-based pagination
Offset-based pagination involves using an offset to specify where to start retrieving data and a limit to specify the number of items to retrieve. It's like saying, "Start from the 10th record and give me the next 5 items". The offset can either be an explicit number or converted from the requested page number. Requesting page 3 with a page size of 5 will translate to an offset of 10 because there are 2 pages before the 3rd page and 2 x 5 = 10. It's more common for an offset-based pagination API to accept the `page` parameter and the server will convert it into the `offset` value when querying the database.

An offset-based pagination API accepts the following parameters:

![06895f25bfa2b2898d8b2f23874a3b79.png](../_resources/06895f25bfa2b2898d8b2f23874a3b79.png)

Given 20 items in a feed, parameters of {size: 5, page: 2} will return items 6 - 10 along with pagination metadata:

```js
{
  "pagination": {
    "size": 5,
    "page": 2,
    "total_pages": 4,
    "total": 20
  },
  "results": [
    {
      "id": "123",
      "author": {
        "id": "456",
        "name": "John Doe"
      },
      "content": "Hello world",
      "image": "https://www.example.com/feed-images.jpg",
      "reactions": {
        "likes": 20,
        "haha": 15
      },
      "created_time": 1620639583
    }
    // ... More posts.
  ]
}
```

and the underlying SQL query resembles:
```sql
SELECT * FROM posts LIMIT 5 OFFSET 0; -- First page
SELECT * FROM posts LIMIT 5 OFFSET 5; -- Second page
```

Offset-based pagination has the following advantages:

- Users can directly jump to a specific page.
- Easy to see the total number of pages.
- Easy to implement on the back end. The `OFFSET` value for a SQL query is calculated using `(page - 1) * size`.
- Easily used with various database systems and not tied to specific data storage mechanisms

However, offset-based pagination comes with some issues:

**Inaccurate page results**: For data that updates frequently, the current page window might be inaccurate after some time. Imagine a user has fetched the first 5 posts in their feed. After sometime, 5 more posts were added. If the users scroll to the bottom of the feed and fetches page 2, the same posts in the original page 1 will be fetched, and the user will see duplicate posts.

```
// Initial posts (newest on the left, oldest on the right)
Posts: A, B, C, D, E, F, G, H, I, J
       ^^^^^^^^^^^^^ Page 1 contains A - E

// New posts added over time
Posts: K, L, M, N, O, A, B, C, D, E, F, G, H, I, J
                      ^^^^^^^^^^^^^ Page 2 also contains A - E

```
Clients can try to be smart and de-duplicate posts by not showing posts that are already visible. However this requires custom logic and the client will have to make a new request to make up for the lack of new posts which costs an extra network roundtrip. For use cases where the number of items can reduce over time, pages can end up missing some items instead.

**Page size cannot be easily changed**: Another downside of offset-based pagination is that the client cannot change the page size for subsequent queries since the offset is a product of the page size and the page being requested.

![42772e6623748a1a103b85ccebd9662d.png](../_resources/42772e6623748a1a103b85ccebd9662d.png)

In the above example, the client will miss out on items 6 and 7 if it goes from `{page: 1, size: 5}` to `{page: 2, size: 7}`.

Query performance degrades over time: Lastly, query performance degrades as the table becomes larger. For huge offsets (e.g. `OFFSET 1000000`) the database still has to read up to those `count` + `offset` rows, discard the offset rows and only return the `count` rows, which results in very poor query performance for large offsets. This is considered back end knowledge but it's useful to know and you might get brownie points for mentioning it.

Offset-based pagination is common in web applications for displaying lists like search results, where jumping to a specific page is required and the results don't update too quickly. Hence blogs, travel booking websites, e-commerce websites will benefit from using offset-based pagination for their search results.

### Cursor-based pagination
Cursor-based pagination uses a pointer (the cursor) to a specific record in a dataset. Instead of saying "give me items 11 to 15", it says "give me 5 items starting after [specific item].".

The cursor is usually a unique identifier, which can be the item id, timestamp, or something else. Subsequent requests use the identifier of the last item as the cursor to fetch the next set of items. In SQL, an example is:

```sql
SELECT * FROM table WHERE id > cursor LIMIT 5.
```

A cursor-based pagination API accepts the following parameters:

![4a830534f486d284f07f89fe9f919326.png](../_resources/4a830534f486d284f07f89fe9f919326.png)

```js
{
  "pagination": {
    "size": 10,
    "next_cursor": "=dXNlcjpVMEc5V0ZYTlo"
  },
  "results": [
    {
      "id": "123",
      "author": {
        "id": "456",
        "name": "John Doe"
      },
      "content": "Hello world",
      "image": "https://www.example.com/feed-images.jpg",
      "reactions": {
        "likes": 20,
        "haha": 15
      },
      "created_time": 1620639583
    }
    // ... More posts.
  ]
}
```

Advantages of cursor-based pagination:

- More efficient and faster on large datasets.
- Avoids the inaccurate page window problem because new posts added over time do not affect the offset, which is determined by a fixed cursor. Great for real-time data.

Downsides of cursor-based pagination:

- Since the client doesn't know the cursor, it cannot jump to a specific page without going through the previous pages.
- Slightly more complex to implement compared to offset-based pagination.

In order for the back end to implement cursor-based pagination, the database needs to uniquely map the cursor to a row, which can be done using a database table's primary key or in some cases, the timestamp.

Facebook, Slack, Zendesk use cursor-based pagination for their developer APIs.

### Which pagination to use for news feed?
In a nutshell, the choice between offset-based pagination and cursor-based pagination largely depends on the data and requirements. Offset-based is simpler and better for static or small datasets where direct access to pages is important. Cursor-based is more efficient and reliable for large, dynamic datasets where the data sequence is important and changes frequently.

For an infinite scrolling news feed where:

- New posts can be added frequently to the top of the feed.
- Newly fetched posts are appended to the end of the feed.
- Table size can grow pretty quickly.

Cursor-based pagination is clearly superior and should be used for a news feed.

Reference: [Evolving API Pagination at Slack](https://slack.engineering/evolving-api-pagination-at-slack/)

### Post creation
This HTTP method is for users to create a new post, which will be shown in their own feed as well as others who they are friends with.

![a1ee29cd0196f77df71e8cd8ecf202f0.png](../_resources/a1ee29cd0196f77df71e8cd8ecf202f0.png)

The parameters of the HTTP API will depend on the type of post made. In most cases, it's not a key discussion point during an interview.

The response format can be either just the single post, or a list of latest posts in the feed. If a single post is returned, the API response will be similar to the feed post item within the feed API:

```js
{
  "id": "124",
  "author": {
    "id": "456",
    "name": "John Doe"
  },
  "content": "Hello world",
  "image": {
    "src": "https://www.example.com/feed-images.jpg",
    "alt": "An image alt" // Either user-provided, or generated on the server.
    // Other useful properties can be included too, such as dimensions.
  },
  "reactions": {
    "likes": 20,
    "haha": 15
  },
  "created_time": 1620639583
}
```

Given this new post data, the client store will need to prepend it to the start of the feed list.

## Optimizations and deep dive

### General optimizations
These optimizations are applicable to every section of the page.

#### Code splitting JavaScript for faster performance
As an application grows, the number of pages and features increase which will result in more JavaScript and CSS code needed to run the application. Code splitting is a technique to split code needed on a page into separate files so that they can be loaded in parallel or when they're needed.

Generally, code splitting can be done on two levels:

- **Split on the page level**: Each page will only load the JavaScript and CSS needed on that page.
- **Lazy loading resources within a page**: Load non-critical resources only when needed or after the initial render, such as code that's only needed lower down on the page, or code that's used only when interacted with (e.g. modals, dialogs).

For a news feed application, there's only a single page, so page-level code splitting is not too relevant, however lazy loading can still be super useful for other purposes. Lazy loading is discussed in more detail under the feed post section as it's most relevant to the feed post UI.

For reference, Facebook splits their JavaScript loading into 3 tiers:

- **Tier 1**: Basic layout needed to display the first paint for the above-the-fold content, including UI skeletons for initial loading states.
- **Tier 2**: JavaScript needed to fully render all above-the-fold content. After Tier 2, nothing on the screen should still be visually changing as a result of code loading.
- **Tier 3**: Resources that are only needed after display that doesn't affect the current pixels on the screen, including logging code and subscriptions for live-updating data.

Source: "[Rebuilding our tech stack for the new Facebook.com](https://engineering.fb.com/2020/05/08/web/facebook-redesign/)" blog post

#### Keyboard shortcuts
Facebook has a number of news feed-specific shortcuts to help users navigate between the posts and perform common actions, super handy! Try it for yourself by hitting the "`Shift` + `?`" keys on facebook.com.

![e976eb3ec20347c39e3794fdf3744401.png](../_resources/e976eb3ec20347c39e3794fdf3744401.png)

#### Error states
Clearly display error states if any network requests have failed, or when there's no network connectivity.

### Feed list optimizations
The feed list refers to the container element that contains the feed post items.

#### Infinite scrolling
An infinite scrolling feed works by fetching the next set of posts when the user has scrolled to the end of their current loaded feed. This results in the user seeing a loading indicator and a short delay where the user has to wait for the new posts to be fetched and displayed.

A way to reduce or entirely eliminate the waiting time is to load the next set of feed posts before the user hits the bottom of the page so that the user never has to see any loading indicators.

A trigger distance of around one viewport height should be sufficient for most cases. The ideal distance is short enough to avoid false positives and wasted bandwidth but also wide enough to load the rest of the contents before the user scrolls to the bottom of the page. A dynamic distance can be calculated based on the network connection speed and how fast the user is scrolling through the feed.

There are two popular ways to implement infinite scroll. Both involve rendering a marker element at the bottom of the feed:

1. **Listen for the `scroll` event**: Add a `scroll` event listener (ideally throttled) to the page or a timer (via `setInterval`) that checks whether the position of the marker element is within a certain threshold from the bottom of the page. The position of the marker element can be obtained using `Element.getBoundingClientRect`.
2. **Intersection Observer API**: Use the [Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) to monitor when the marker element is entering or exiting another element or intersecting by a specified amount.

The Intersection Observer API is a browser native API and is preferred over `Element.getBoundingClientRect()`.

> *The Intersection Observer API lets code register a callback function that is executed whenever an element they wish to monitor enters or exits another element (or the viewport), or when the amount by which the two intersect changes by a requested amount. This way, sites no longer need to do anything on the main thread to watch for this kind of element intersection, and the browser is free to optimize the management of intersections as it sees fit.*

### Virtualized lists
With infinite scrolling, all the loaded feed items are on one single page. As the user scrolls further down the page, more posts are appended to the DOM and with feed posts having complex DOM structure (lots of details to render), the DOM size rapidly increases. As social media websites are long-lived applications (especially if single-page app) and the feed items list can easily grow really long quickly, the number of feed items can be a cause of performance issues in terms of DOM size, rendering, and browser memory usage.

Virtualized lists is a technique to render only the posts that are within the viewport. In practice, Facebook replaces the contents of off-screen feed posts with empty `<div>`s, add dynamically calculated inline styles (e.g. `style="height: 300px`") to set the height of the posts so as to preserve the scroll position and add the `hidden` [attribute](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/hidden) to them. This will improve rendering performance in terms of:

- **Browser painting**: Fewer DOM nodes to render and fewer layout computations to be made.
- **Virtual DOM reconciliation (React-specific)**: Since the post is now a simpler empty version, it's easier for React (the UI library that Facebook is using to render the feed) to diff the virtual DOM vs the real DOM to determine what DOM updates have to be made.

Both Facebook and Twitter websites use virtualized lists.

### Loading indicators
For users who scroll really fast, even though the browser kicks off the request for the next set of posts before the user even reaches the bottom of the page, the request might not have returned yet and a loading indicator should be shown to reflect the request status.

Rather than showing a spinner, a better experience would be to use a [shimmer loading effect](https://docs.flutter.dev/cookbook/effects/shimmer-loading) which resembles the contents of the posts. This looks more aesthetically pleasing and can be used also reduce layout thrash after the new posts are loaded.

### Dynamic loading count
As mentioned above in the "Interface" section, cursor-based pagination is more suitable for a news feed. A benefit of cursor-based pagination is that the client can change how many items to fetch in subsequent calls. We can use this to our advantage by customizing the number of posts to load based on the browser window height.

For the initial load, we do not know the window height so we need to be conservative and overfetch the number of posts needed. But for subsequent fetches, we know the browser window height and can customize the number of posts to fetch based on that.

### Preserving feed scroll position on remounting
Feed scroll positions should be preserved if users navigate to another page and back to the feed. This can be achieved in single-page applications if the feed list data is cached within the client store along with the scroll position. When the user goes back to the feed page, since the data is already on the client, the feed list can be read from the client store and immediately presented on the screen with the previous scroll position; no server round-trip is needed.

### Stale feeds
It's not uncommon for users to leave their news feed application open as a browser tab and not refresh it at all. It'd be a good idea to prompt the user to refresh or refetch the feed if the last fetched timestamp is more than a few hours ago, as there can be new posts and the loaded feed is considered stale. When a new feed is refetched, the current feed can be entirely removed from memory to free up memory space.

Another approach is to automatically append new feed posts to the top of the feed, but that might not be desired and extra care has to be made in order not to affect the scroll position.

As of writing, Facebook forces a feed refresh and scrolls to the top if the tab has been left open for some duration.

### Feed post optimizations
The feed post refers to the individual post element that contains details of the post: author, timestamp, contents, like/commenting buttons.

#### Delivering data-driven dependencies only when needed
News feed posts can come in many different formats (text, image, videos, polls, etc) and each post requires custom rendering code. In reality, Facebook feed supports over 50 different post formats!

One way to support all the post formats on the client is to have the client load the component JavaScript code for all possible formats upfront so that any kind of feed post format can be rendered. However, not all users' feed will contain all the post formats and and there will likely be a lot of unused JavaScript. With the large variety of feed post formats, loading the JavaScript code for all of them upfront is sure to cause performance issues.

If only we could lazy load components depending on the data received! That's already possible but requires an extra network round-trip to lazy load the components after the data is fetched and we know the type of posts rendered.

Facebook fetches data from the server using Relay, which is a JavaScript-based GraphQL client. Relay couples React components with GraphQL to allow React components to declare exactly which data fields are needed and Relay will fetch them via GraphQL and provide the components with data. Relay has a feature called [data-driven dependencies](https://relay.dev/docs/glossary/#match) via the `@match` and `@module` GraphQL directives, which fetches component code along with the respective type of data, effectively solving the excess components problem mentioned above without an extra network round-trip. You only load the relevant code when a particular format for a post is being shown.

```graphql
// Sample GraphQL query to demonstrate data-driven dependencies.
... on Post {
  ... on TextPost {
    @module('TextComponent.js')
    contents
  }
  ... on ImagePost {
    @module('ImageComponent.js')
    image_data {
      alt
      dimensions
    }
  }
}
```

The above GraphQL query tells the back end to return the `TextComponent` JavaScript code along with the text contents if the post is a text-based post and return the `ImageComponent` JavaScript code along with the image data if the post has image attachments. There is no need for the client to load component JavaScript code for all the possible post formats upfront, reducing the initial JavaScript needed on the page.

### Rendering mentions/hashtags
You may have noticed that textual content within feed posts can be more than plain text. For social media applications, it is common to see mentions and hashtags.

![dd5b39bf15f88639a6317b8bfd7f3776.png](../_resources/dd5b39bf15f88639a6317b8bfd7f3776.png)

In Stephen Curry's post above, see that he used the "#AboutLastNight" hashtag and mentioned the "HBO Max" Facebook page. His post message has to be stored in a special format such that it contains metadata about these tags and mentions.

What format should the message be so that it can store data about mentions/hashtags? Let's discuss the possible formats and their pros and cons.

HTML format: The simplest format is HTML, you store the message the way you want it to be displayed.

```html
<a href="...">#AboutLastNight</a> is here... and ready to change the meaning of date night...

Absolute comedy 🤣 Dropping 2/10 on <a href="...">HBO Max</a>!
```

Storing as HTML is usually bad because there's a potential for causing a Cross-Site Scripting (XSS) vulnerability. Also, in most cases it's better to decouple the message's metadata from displaying, perhaps in future you want to decorate the mentions/hashtags before rendering and want to add classnames to the links. HTML format also makes the API less reusable on non-web clients (e.g. iOS/Android).

**Custom syntax**: A custom syntax can be used to capture metadata about hashtags and mentions.

- **Hashtags**: Hashtags don't actually need a special syntax, words that start with a "#" can be considered a hashtag.
- **Mentions**: A syntax like `[[#1234: HBO Max]]` is sufficient to capture the entity ID and the text to display. It's not sufficient to just store the entity ID because sites like Facebook allow users to customize the text within the mention.

Before rendering the message, the string can be parsed for hashtags and mentions using regex and replaced with custom-styled links. Custom syntax is a lightweight solution which is robust enough if you don't anticipate new kinds of rich text entities to be supported in future.

Rich text editor format: Draft.js is a popular rich text editor by Meta for composing rich text. Draft.js allows users to extend the functionality and create their own rich text entities such as hashtags and mentions. It defines a custom Draft.js editor state format which is being used by the Draft.js editor. In 2022, Meta released Lexical, a successor to Draft.js and is using Lexical for rich text editing and displaying rich text entities on facebook.com. The underlying formats are similar and we will discuss Draft.js' format.

Draft.js is just one example of a rich text format, there are many out there to choose from. The editor state resembles an Abstract Syntax Tree and can be serialized into a JSON string to be stored. The benefits of using a popular rich text format is that you don't have to write custom parsing code and can easily extend to more types of rich text entities in future. However, these formats tend to be longer strings than a custom syntax version and will result in larger network payload size and require more disk space to store.

An example of how the post above can be represented in Draft.js.

```js
{
  content: [
    {
      type: 'HASHTAG',
      content: '#AboutLastNight',
    },
    {
      type: 'TEXT',
      content: ' is here... and ready to change ... Dropping 2/10 on ',
    },
    {
      type: 'MENTION',
      content: 'HBO Max',
      entityID: 1234,
    },
    {
      type: 'TEXT',
      content: '!',
    },
  ];
}
```

### Rendering images
Since there can be images in a feed post, we can also briefly discuss some image optimization techniques:

- **Content Delivery Network (CDN)**: Use a (CDN) to host and serve images for faster loading performance.
- **Modern image formats**: Use modern image formats such as [WebP](https://developers.google.com/speed/webp) which provides superior lossless and lossy image compression.
-  **`<img>`s should use proper `alt` text**
   - Facebook provides `alt` text for user-uploaded images by using Machine Learning and Computer
   - Vision to process the images and generate a description. Generative AI models are also very good at doing that these days.
- Image loading based on device screen properties
	- Send the browser dimensions in the feed list requests so that server can decide what image size to return.
	- Use `srcset` if there are image processing (resizing) capabilities to load the most suitable image file for the current viewport.
- Adaptive image loading based on network speed
	- **Devices with good internet connectivity/on WiF**i: Prefetch offscreen images that are not in the viewport yet but about to enter viewport.
	- **Poor internet connection**: Render a low-resolution placeholder image and require users to explicitly click on them to load the high-resolution image.

### Lazy load code that is not needed for initial render
Many interactions with a feed post are not needed on initial render:

- Reactions popover.
- Dropdown menu revealed by the top-right ellipsis icon button, which is usually meant to conceal additional actions.

The code for these components can be downloaded when:

- The browser is idle as a lower-priority task.
- On demand, when the user hovers over the buttons or clicks on them.

These are considered Tier 3 dependencies going by Facebook's tier definition above.

### Optimistic updates 
Optimistic update is a performance technique where the client immediately reflect the updated state after a user interaction that hits the server and optimistically assume that the server request succeeds, which should be the case for most requests. This gives users instant feedback and improves the perceived performance. If the server request fails, we can revert the UI changes and display an error message.

For a news feed, optimistic updates can be applied for reaction interactions by immediately showing the user's reaction and an updated total count of the reactions.

Optimistic updates is a powerful feature built into modern query libraries like Relay, SWR and React Query.

### Timestamp rendering
Timestamp rendering is a topic worth discussing because of a few issues: multilingual timestamps and stale relative timestamps.

**Multilingual timestamps**: Globally popular sites like Facebook and Twitter have to ensure their UI works well for different languages. There are a few ways to support multilingual timestamps:

1. **Server returns the raw timestamp**: Server returns the raw timestamp and the client renders in the user's language. This approach is flexible but requires the client to contain the grammar rules and translation strings for different languages, which can amount to significant JavaScript size depending on the number of supported languages,
2. **Server returns the translated timestamp**: This requires processing on the server, but you don't have to ship timestamp formatting rules for various languages to the client. However, since translation is done on the server, clients cannot manipulate the timestamp on the client.
3. **Intl API**: Modern browsers can leverage Intl.DateTimeFormat() and Intl.RelativeTimeFormat() to transform a raw timestamp into translated datetime strings in the desired format.

```js
const date = new Date(Date.UTC(2021, 11, 20, 3, 23, 16, 738));
console.log(
  new Intl.DateTimeFormat('zh-CN', {
    dateStyle: 'full',
    timeStyle: 'long',
  }).format(date),
); // 2021年12月20日星期一 GMT+8 11:23:16

console.log(
  new Intl.RelativeTimeFormat('zh-CN', {
    dateStyle: 'full',
    timeStyle: 'long',
  }).format(-1, 'day'),
); // 1天前

```

**Relative timestamps can turn stale**: If timestamps are displayed using a relative format (e.g. 3 minutes ago, 1 hour ago, 2 weeks ago, etc), recent timestamps can easily go stale especially for applications where users don't refresh the page. A timer can be used to constantly update the timestamps if they are recent (less than an hour old) such that any significant time that has passed will be reflected correctly.

### Icon rendering
Icons are needed within the action buttons of a post for liking, commenting, sharing, etc. There are a few ways to render icons:

![fbe633f6239ae257bec09feb7cbb9859.png](../_resources/fbe633f6239ae257bec09feb7cbb9859.png)

Facebook and Twitter use inlined SVGs and that also seems to be the trend these days. This technique is not specific to news feeds, it's relevant to almost every web app.

### Post truncation
Truncate posts which have super long message content and reveal the rest behind a "See more" button.

For posts with large amount of activity (e.g. many likes, reactions, shares), abbreviate counts appropriately instead of rendering the raw count so that it's easier to read and the magnitude is still sufficiently conveyed:

- **Good**: John, Mary and 103K others
- **Bad**: John, Mary and 103,312 others

This summary line can be either constructed on the server or the client. The pros and cons of doing on the server vs the client is similar to that of timestamp rendering. However, you should definitely not send down the entire list of users if it's huge as it's likely not needed or useful.

#### Feed comments
If time permits, we can discuss how the feed comments can be built. In general, the same rules apply to comments rendering and comment drafting:

- Cursor-based pagination for fetching the list of comments.
- Drafting and editing comments can be done in a similar fashion as drafting/editing posts.
- Lazy load emoji/sticker pickers in the comment inputs.
- Optimistic updates
	- Immediately reflect new comments by appending the user's new comment to the existing list of comments.
	- Immediately display new reactions and updated reaction counts.

### Live comment updates
Live updates for Facebook feed comments enhance user engagement and interaction by providing real-time visibility of new comments and updated reaction counts. This fosters a dynamic and responsive communication environment, encouraging users to actively participate in ongoing conversations without the need for manual refreshing. The immediacy of live updates contributes to increased user retention.

The common ways to implement live updates on a client include:

- **Short polling**: Short polling is a technique in which the client repeatedly sends requests to the server at fixed intervals to check for updates. The connection is closed after each request, and the server responds immediately with the current state or any available updates. While short polling is straightforward to implement, it may result in higher network traffic and server load compared to the more advanced techniques mentioned below.
- **Long polling**: Long polling extends on the idea of short polling by keeping the connection open until new data is available. While simpler to implement, it may introduce latency and increased server load compared to other approaches.
- **Server-Sent Events (SSE)**: SSE is a standard web technology that enables servers to push updates to web clients over a single HTTP connection. It's a simple and efficient mechanism for real-time updates, particularly well-suited for scenarios where updates are initiated by the server.
- **WebSockets**: WebSockets provide a full-duplex communication channel over a single, long-lived connection. This bidirectional communication allows both the server and the client to send messages to each other at any time. WebSockets are suitable for applications that require low latency and high interactivity.
- **HTTP/2 server push**: With HTTP/2, the server can push updates to the client without waiting for the client to request them. While HTTP/2 server push is not as widely used as other methods for live updates, it can be an efficient solution in certain scenarios.

Facebook uses WebSockets for live updates on the website.

While showing live updates is great, it is not efficient to be fetching updates for posts that have gone out of view in the feed. Clients can subscribe/unsubscribe to updates for posts based on whether the post is visible, which lightens the load on server infrastructure.

Additionally, not all posts should be treated equally. Posts by users with many followers (e.g. celebrities and politicians) will be seen by many more people and hence more likely to receive an update. For such posts, new comments and reactions will be frequently added/updated and it will not be wise to fetch every new post or reaction since the update rate is too high for the user to read every new comment. Hence for such posts, the live updates can be debounced/throttled. Beyond a certain threshold, just fetching the updated comments and reactions count will be sufficient.

### Feed composer optimizations
#### Rich text for hashtags and mentions
When drafting a post in the composer, it'd be a nice to have an WYSIWYG editing experience look like the result which contain hashtags and mentions. However, `<input>`s and `<textarea>`s only allows input and display of plain text. The `contenteditable` attribute turns an element into an editable rich text editor.


### Lazy load dependencies
Like rendering news feed posts, users can draft posts in many different formats which require specialized rendering code per format. Lazy loading and can be used to load the resources for the desired formats and optional features in an on-demand fashion.

Non-crucial features where the code can be lazy loaded on demand:

- Image uploader
- GIF picker
- Emoji picker
- Sticker picker
- Background images

### Accessibility
Here are some accessibility considerations for a news feed.

#### Feed list
Add `role="feed"` to the feed HTML element.

#### Feed posts
- Add `role="article"` to each feed post HTML element.
- `aria-labelledby="<id>"` where the HTML tag containing the feed author name has that `id` attribute.
- Contents within the feed post should be focusable via keyboards (add `tabindex="0"`) and the appropriate `aria-role`.

#### Feed interactions
On Facebook website, users can have more reaction options by hovering over the "Like" button. To allow the same functionality for keyboard users, Facebook displays a button that only appears on focus and the reactions menu can be opened via that button.
Icon-only buttons should have `aria-labels` if there are no accompanying labels (e.g. Twitter).

## References
- [Rebuilding our tech stack for the new Facebook.com](https://engineering.fb.com/2020/05/08/web/facebook-redesign/)
- [Making Facebook.com accessible to as many people as possible](https://engineering.fb.com/2020/07/30/web/facebook-com-accessibility/)
- [How we built Twitter Lite](https://blog.x.com/engineering/en_us/topics/open-source/2017/how-we-built-twitter-lite)
- [Building the new Twitter.com](https://blog.x.com/engineering/en_us/topics/infrastructure/2019/buildingthenewtwitter)
- [Dissecting Twitter's Redux Store](https://medium.com/statuscode/dissecting-twitters-redux-store-d7280b62c6b1)
- [Twitter Lite and High Performance React Progressive Web Apps at Scale](https://medium.com/@paularmstrong/twitter-lite-and-high-performance-react-progressive-web-apps-at-scale-d28a00e780a3)
- [Making Instagram.com faster: Part 1](https://instagram-engineering.com/making-instagram-com-faster-part-1-62cc0c327538)
- [Making Instagram.com faster: Part 2](https://instagram-engineering.com/making-instagram-com-faster-part-2-f350c8fba0d4)
- [Making Instagram.com faster: Part 3 — cache first](https://instagram-engineering.com/making-instagram-com-faster-part-3-cache-first-6f3f130b9669)
- [Evolving API Pagination at Slack](https://slack.engineering/evolving-api-pagination-at-slack/)









