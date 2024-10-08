# Pinterest

Masonry layout is a popular and flexible grid-based layout design. Unlike traditional grids, where elements in the same row have the same height, elements are placed within columns but can have varying heights, resulting in a more organic and visually interesting arrangement.

Such a layout uses space efficiently as it fills up most of the available screen estate with its contents and leaves little white space between elements. It is commonly used for presenting user-generated media like images and GIFs. The most famous site with this layout is Pinterest.

## Question

Design the Pinterest homepage, with a focus on the masonry layout.

Pinterest front end system design questions are commonly asked in two manners:

1. Design the Pinterest homepage which covers page architecture, masonry layout, data fetching, etc.
2. Design a masonry component, discussing the props, layout approach, etc.

We will focus on the former but also provide enough content and guidance for the latter. In fact, the actual masonry component Pinterest uses on their homepage is [built in React and open sourced](https://gestalt.pinterest.systems/web/masonry)! You can dive into the source code to know the ins and outs of the component and also use it on your own sites.

**Note**: Pinterest is essentially an image feed with a multi-column layout. Hence it shares many similarities with the News Feed system design and Instagram system design. Please have a read of those questions before starting on this. For this question, the discussion will be centered around the masonry layout and less about general feed optimizations.

## Requirements exploration

### What are the core features to be supported?
- Masonry layout of feed items (pins).
- More items should be loaded in as the user scrolls down.

### How should the pins be ordered?
As much as possible, the pins should be placed respecting their position within the feed, i.e. pins that are at the front of the returned results should appear higher on the page.

### What kind of pins/items are supported?
Focus on just images. Exclude videos and GIFs.

### What devices will the application be used on?
Primarily desktop, but should be usable on mobile and tablet as well.

## Glossary
We'll refer to "feed items" as "pins" and use them interchangeably below. A pin contains basic metadata such as the image, subtitle, etc.

## Architecture / high-level design

### Server-side rendering (SSR) or Client-side rendering (CSR)?
For a website like Pinterest, which is known for its visual content and user interactivity, a hybrid approach might be a good choice. Pinterest has a complex user interface with endless scrolling and dynamic content. Initially, you might use SSR for the landing page and critical content (for SEO and fast initial load), and then switch to CSR for interactivity as users browse pins and boards.

In reality, Pinterest uses a hybrid rendering strategy that involves SSR with hydration. The initial pins markup and data are included in the initial HTML, but without any positioning data. This will be covered in more detail below.

### Single-page application (SPA) or Multi-page application (MPA)?
In feed applications, it's a common UX for the full contents of a pin to be opened within a modal, overlaying the feed. When the user closes the modal, users can continue scrolling the feed from where they left off. This UX is present on Facebook and Instagram, as well as Pinterest.

Hence it's crucial to use SPAs, at least for the feed and the pin details routes so that browsing individual pins can use a client-side navigation to the pin details route. In MPAs, a full page navigation will destroy the current's page DOM and feed data stored in-memory, causing the scroll position of the previous page to be lost if the user clicks the "Back" button.

### Component responsibilities

![9a813cc011dd5611caebbd203297469e.png](../_resources/9a813cc011dd5611caebbd203297469e.png)

The architecture diagram of a Pinterest feed is relatively simple since we are only focusing on the feed and layout.

- **Server**: Provides HTTP APIs to fetch the feed of pins and also subsequent pages of pins as the user scrolls through the feed.
- **Client store**: Stores data needed across the whole application. For this question there's a list of pins to be stored.
- **Homepage**: Displays the list of pins.
	- Masonry component: UI component that takes in a list of pins and displays them in a masonry layout.

**Note**: Conventionally, we have "controller" and "client store" as separate entities, but since the types of data involved and data fetching required for the feed is fairly limited, we can combine the controller into the client store and make the client store take up data fetching responsibilities. However, in practice there are many more state values and interactions to handle and a separation would be beneficial.

## Data model

The homepage feed shows a list of pins fetched from the server, hence most of the data involved in this application will be server-originated data. These pins data will be augmented with positioning data used by the masonry layout.

![281f447f1e14a79dc1a01fa9d262fce8.png](../_resources/281f447f1e14a79dc1a01fa9d262fce8.png)

Although the `Feed` and `Pin` entities belong to the homepage, all server-originated data can be stored in the client store and queried by components which need them. E.g. for a pin details page, assuming no additional data is needed, it can read the pin details from the client store and display additional details such as the author, title, description.

The shape of the client store is not particularly important here, as long as it is in a format that can be easily retrieved from the components. A normalized store like the one mentioned in News Feed system design will allow efficient lookup via the pin ID. New pins fetched from the second page should be joined with the previous pins into a single list with the pagination parameters (`cursor`) updated.

### Pinterest-specific data
Due to the masonry layout requirements of Pinterest, Pins include additional metadata that enables the layout and an improved user experience like:

- **Image dimensions (height & width)**: So that we can use the data for calculating layout without having to first load the image.
- **Ordering**: Where to place the pin. Details covered in the section below on "implementing masonry layout".
- **Responsive image dimensions**: This is a list of image URLs and their corresponding sizes, to facilitate responsive and performant masonry layout.
- **Pin state**: Whether a pin's image is loaded, painted/displayed, or errored. More details below.
- **Dominant color (hex value)**: The dominant color to be used as the background for the placeholder while the image loads.

## Interface definition (API)
For this question, all we need is a single HTTP API for fetching the feed data:

![6fe4227f5027167b49e65a8417f02a91.png](../_resources/6fe4227f5027167b49e65a8417f02a91.png)

#### Sample response

```js
{
  "pagination": {
    "size": 20,
    "next_cursor": "=dXNlcjpVMEc5V0ZYTlo"
  },
  "pins": [
    {
      "id": 123, // Pin ID.
      "created_at": "Sun, 01 Oct 2023 17:59:58 +0000",
      "alt_text": "Pixel art turnip",
      "dominant_color": "#ffd4ec",
      "image_url": "https://www.greatcdn.com/img/941b3f3d917f598577b4399b636a5c26.jpg"
      // In practice, the images payload is more complex, see below.
      // More metadata is also included.
    }
    // ... More pins.
  ]
}
```

In practice, Pinterest's feed API doesn't include a size parameter and always returns 25 pins. If you're interested to see the full feed payload for yourself:

- Visit https://pinterest.com in your browser.
- Open the network tab.
- Scroll down to fetch the next page of feed data.
- Filter for the request URL starting with ["https://www.pinterest.com/resource/UserHomefeedResource/get"](https://www.pinterest.com/resource/UserHomefeedResource/get/).

#### Pagination approach
For infinite scrolling feeds there isn't a need to jump to a specific page, hence cursor-based pagination can be used here for reasons similar to News Feed system design.

## Optimizations and deep dive
For conventional feeds, a reasonable flow for displaying feed images is as follows:

- **Load data**: Fetch feed data containing a list of feed items from the server.
- **Render**: Display the image by adding an `<img>` tag to the DOM.
- **Image loading + Painting**: The browser downloads the image from the CDN and paints it to the screen.

However for Pinterest, the flow is a bit different due to the following:

- **Multi-column layout**: The multi-column layout makes things more complex than a single-column news feed. Traditional CSS layouts using flex and grid aren't well-suited for producing masonry layouts.
- **Multiple images present on-screen**: Many images are present on the screen at once. It'd be a poor experience for the images to be painted out-of-order.

Taking these into account, the most important aspects of a Pinterest homepage feed to dive into are:

- Resource loading (feed and media).
- Masonry layout implementation.
- Performance.
- An advanced technique, paint scheduling will also be discussed.

There are two types of resources to be loaded: feed item (pins) data and each pin's images. For a fast initial load and paint, Pinterest does the following optimizations:

1. The homepage is SSR-ed, meaning the initial HTML returned by the server already contains the near-final markup and includes the pins' `<img>` tags. Because of this, there's no need for the client to make a client-side request to the feed API and fetch pins data over the network. The pins data can be serialized into JSON and injected into the client store.
2. Preload images using `<link rel="preload">`.

### Feed loading
Beyond the initial page, when the user scrolls to the bottom of the loaded items, the next page of feed items has to be loaded and then displayed. This experience is called infinite scrolling and are two ways to trigger the loading:

1. Load the next page when the user has reached the bottom of the list. This is bad because the user has to wait for next page of items to be loaded, and then for the next page's images to be loaded, before the images are visible.
2. Load the next page when the user is near the bottom of the list, before the user reaches the bottom of the list. By doing this, the next page and their images will likely have already been loaded and can be shown immediately. The user does not have to wait for the image to load and won't see any loading states.

#### Dynamic page size
When fetching feed items, Pinterest uses a page size of 24 regardless of device, but this can be improved. For large displays, the feed API is requested very often as the user quickly reaches the end of the current feed. To prevent this, clients can specify a different value for `size` depending on the device dimensions:

![1db558afcd0679c5797fba3faa1024a3.png](../_resources/1db558afcd0679c5797fba3faa1024a3.png)

A plausible heuristic is to fetch around two to three screens worth of pins each time.

#### Infinite scrolling
The common ways to implement infinite scrolling has been covered in News Feed system design.

### Media loading

#### `<img>` tag attributes

Pinterest uses `<img>` tags with the following attributes:

- `alt`: Text description of the image, used when images fail to load and for screen readers.
- `fetchpriority`: Provides a hint of the relative priority to use when fetching the image. It's not a standard yet. Pinterest uses fetchpriority="auto"
- `loading`: Indicates how the browser should load the image. Pinterest uses loading="auto", which is the default value and not sure why it's necessary. Possible values include "lazy", which defers the load until the image or iframe reaches a distance threshold from the viewport.
- `src`: The image URL.
- `srcset`: Enables multiple image sources, each with different resolutions or sizes.

#### Responsive images
As mentioned above, the `srcset` attribute allows the browser to choose the most appropriate image source to display based on the user's device's screen size and resolution. It is a part of responsive web design techniques and helps optimize the loading of images for different screen sizes, which can enhance the user experience and save bandwidth.

In reality, Pinterest's feed API returns images in the following format and not just a single `image_url` string:

```js
{
  "id": "809944314208458040",
  "alt_text": "Year End Sale Font Bundle",
  "images": {
    "170x": {
      "width": 170,
      "height": 113,
      "url": "https://i.pinimg.com/170x/b5/5c/de/b55cde4a2ec2f7827b2deac1312cbd93.jpg"
    },
    "236x": {
      "width": 236,
      "height": 157,
      "url": "https://i.pinimg.com/236x/b5/5c/de/b55cde4a2ec2f7827b2deac1312cbd93.jpg"
    },
    "474x": {
      "width": 474,
      "height": 316,
      "url": "https://i.pinimg.com/474x/b5/5c/de/b55cde4a2ec2f7827b2deac1312cbd93.jpg"
    },
    "736x": {
      "width": 736,
      "height": 491,
      "url": "https://i.pinimg.com/736x/b5/5c/de/b55cde4a2ec2f7827b2deac1312cbd93.jpg"
    },
    "orig": {
      "width": 1160,
      "height": 774,
      "url": "https://i.pinimg.com/originals/b5/5c/de/b55cde4a2ec2f7827b2deac1312cbd93.jpg"
    }
  }
  // Other data fields omitted.
}
```

And results in such an `<img>` element:

```html
<img
  alt="This contains an image of: Year End Sale Font Bundle"
  fetchpriority="auto"
  loading="auto"
  src="https://i.pinimg.com/236x/b5/5c/de/b55cde4a2ec2f7827b2deac1312cbd93.jpg"
  srcset="
    https://i.pinimg.com/236x/b5/5c/de/b55cde4a2ec2f7827b2deac1312cbd93.jpg      1x,
    https://i.pinimg.com/474x/b5/5c/de/b55cde4a2ec2f7827b2deac1312cbd93.jpg      2x,
    https://i.pinimg.com/736x/b5/5c/de/b55cde4a2ec2f7827b2deac1312cbd93.jpg      3x,
    https://i.pinimg.com/originals/b5/5c/de/b55cde4a2ec2f7827b2deac1312cbd93.jpg 4x
  " />
```

An alternative approach to achieve responsive images is to use the `<picture>` tag. The differences between `<img srcset="...">` or `<picture>` are subtle and for most cases you can use either. The article "[Picture Tags vs Img Tags. Their Uses and Misuses](https://medium.com/@truszko1/picture-tags-vs-img-tags-their-uses-and-misuses-4b4a7881a8e1)" does a decent job covering the differences.

### Image preloading
The images present in the initial load are preloaded via `<link rel="preload">`. This tells the browser about critical resources that should be loaded as soon as possible, before they are discovered in HTML. This is especially useful for resources that are not easily discoverable, such as fonts included in stylesheets, background images, or resources loaded from a script. [Read more about this on web.dev](https://web.dev/articles/preload-responsive-images).

```html
<!-- Sample extracted from Pinterest's HTML -->
<link
  rel="preload"
  nonce="7deba0d15af118e95df9c836e67724dc"
  href="https://i.pinimg.com/236x/83/84/3a/83843ab4e2cbdea8b99ead3e1f0654d1.jpg"
  imagesrcset="https://i.pinimg.com/236x/83/84/3a/83843ab4e2cbdea8b99ead3e1f0654d1.jpg 1x, https://i.pinimg.com/474x/83/84/3a/83843ab4e2cbdea8b99ead3e1f0654d1.jpg 2x, https://i.pinimg.com/736x/83/84/3a/83843ab4e2cbdea8b99ead3e1f0654d1.jpg 3x, https://i.pinimg.com/originals/83/84/3a/83843ab4e2cbdea8b99ead3e1f0654d1.jpg 4x"
  as="image" />
<link
  rel="preload"
  nonce="7deba0d15af118e95df9c836e67724dc"
  href="https://i.pinimg.com/236x/02/f8/ac/02f8acb5e46eaa42a909f9be862f519b.jpg"
  imagesrcset="https://i.pinimg.com/236x/02/f8/ac/02f8acb5e46eaa42a909f9be862f519b.jpg 1x, https://i.pinimg.com/474x/02/f8/ac/02f8acb5e46eaa42a909f9be862f519b.jpg 2x, https://i.pinimg.com/736x/02/f8/ac/02f8acb5e46eaa42a909f9be862f519b.jpg 3x, https://i.pinimg.com/originals/02/f8/ac/02f8acb5e46eaa42a909f9be862f519b.png 4x"
  as="image" />
<!-- Preload 10 images in total -->

```

### Progressive JPEGs
Pin images are served as [Progressive JPEGs](https://www.hostinger.com/tutorials/website/improving-website-performance-using-progressive-jpeg-images), which improve image quality with each scan. The conventional JPEG format (called baseline JPEG) loads images sequentially, rendering them line by line from top to bottom, with each line being pixel-perfect. As a result, it may take some time for the entire image to load completely.

On the other hand, with progressive JPEG, the entire image initially appears as a single entity, albeit in a blurry and pixelated state. Over time, it progressively sharpens and refines until a clear, fully loaded image emerges.

### Media formats
There are multiple image formats to use, each with their own pros and cons. The general recommendation these days is use the [WebP format by Google](https://developers.google.com/speed/webp) if browser compatibility is not the highest priority.

Pinterest uses the JPEG format for images likely due to JPEGs having wider browser support (WebP isn't supported on IE11 and older versions of Safari) and because progressive JPEGs already provide a great loading experience. Pinterest's audience is the average consumer and hence browser support is crucial.

### Layout and rendering
With feed and media loading covered, we can discuss how to present these data in a masonry layout and in an order that provides the best user experience.

#### Masonry layout implementation
A masonry layout is notoriously difficult to implement. Achieving the signature "brick wall" or staggered layout of masonry requires precise positioning of items. This can be complex, especially when we want items to fit together neatly and utilize the available space optimally. Traditional CSS layouts using flex and grid aren't well-suited for producing masonry layouts.

There are two popular ways to achieve a masonry layout on the web:

- Rows of columns
- Absolute positioning

**Row of columns**: This method involves rendering equal-width columns then placing items within each column. This approach heavily leverages the browser for positioning.

The pros of this approach are that it is easy to implement as it leverages `display: flex` and the required CSS is not very complex. If the height of any item changes, the positions of the items within that column will be updated automatically.

However, one huge downside is that the DOM order is now column-first. Imagine a keyboard user wants to get to the top pin in the rightmost column. With such a DOM:

```html
<div class="container">
  <div class="column">
    <!-- Focus is currently here -->
    <div class="item">1</div>
    <div class="item">2</div>
    <div class="item">3</div>
    <div class="item">4</div>
  </div>
  <div class="column">
    <div class="item">5</div>
    <div class="item">6</div>
    <div class="item">7</div>
    <div class="item">8</div>
    <div class="item">9</div>
  </div>
  <div class="column">
    <!-- Desired pin requires pressing the "Tab" key 9 times -->
    <div class="item">10</div>
    <div class="item">11</div>
    <div class="item">12</div>
    <div class="item">13</div>
  </div>
</div>
```

The keyboard user has to tab through two columns of pins before finally reaching that item. This is counter-intuitive and frustrating when the desired pin is right at the top. This layout approach results in an inefficient DOM order for keyboard users, and is a deal-breaker.

The following example is implemented using the "row of columns" layout and the numbers within the items indicate the elements tabbing order.

[Sandbox example](https://codesandbox.io/p/sandbox/pinterest-layout-row-of-columns-f7s8vt?from-embed)

**Absolute positioning**: A better approach is to calculate exactly where the pins should be positioned within a `position: relative` parent container and place them exactly where they should be via `position: absolute; top: Y; left: X`.

The following diagram demonstrates the `top` and `left` CSS style values of the items for a 3-column layout container.

```html
<div class="container">
  <div class="item" style="height: 250px; top: 0px; left: 0px;">1</div>
  <div class="item" style="height: 300px; top: 0px; left: 80px;">2</div>
  <div class="item" style="height: 110px; top: 0px; left: 160px;">3</div>
  <div class="item" style="height: 200px; top: 260px; left: 0px;">4</div>
  <div class="item" style="height: 70px; top: 310px; left: 80px;">5</div>
  <div class="item" style="height: 330px; top: 120px; left: 160px;">6</div>
</div>
```
![94a465d1124a548901de320617c293a7.png](../_resources/94a465d1124a548901de320617c293a7.png)

With absolute positioning, the DOM order of the items does not affect the visual result at all. Hence we are free to arrange the DOM in the way the tabbing order should be.

The following example is implemented using the absolute positioning approach and the numbers within the items indicate the elements' tabbing order. Notice that there's only one layer of children nodes within the container; there's no nesting involved and the markup is very clean.

[Sandbox example](https://codesandbox.io/p/sandbox/pinterest-layout-position-absolute-dqtrlk?from-embed)

Typically, `absolute`-positioned elements are taken out of the normal document flow and usually do not affect the positioning or layout of other elements on the page. As a result, changes to these elements, such as their size or position, usually do not trigger a browser reflow. However, adding more pins to the bottom of the container, even though they are `absolute`-ly positioned, will increase the height of the container and result in reflows.

Another advantage of `absolute` positioning is that list virtualization is easier to implement, more details below.

The downside of this approach is that the client has to write code to calculate where the pins should be as opposed to the "rows of columns" approach where the browser does it for us. Beyond the initial calculations, calculations will need to be done again for the whole feed if the window resizes past the breakpoints such that the number of columns is different.

Additionally, if any item's height changes, items below it will not be repositioned unless calculations are performed manually for the affected items and their positions are also updated.

By default, the container isn't fluid and has a fixed width for a few predefined breakpoints. Even if the browser resizes slightly, as long as the number of columns remain the same, the calculated positions can still be used.

In practice, Pinterest uses CSS transforms (e.g. `transform: translateY(100px) translateX(100px))` instead of using `top: 100px`and `left: 100px`, presumably because it is more performant to use CSS transforms. CSS transforms such as `translate`, `scale` and `rotate` are typically hardware-accelerated by modern web browsers. This means that the browser offloads the rendering of transformed elements to the GPU (Graphics Processing Unit), which can handle these operations more efficiently. Absolute positioning, on the other hand, doesn't always benefit from GPU acceleration to the same extent.

The general approach is the same though – the items are still positioned relative to the top left corner of the pins container.

Pinterest's open source Masonry React component [renders the items offscreen to measure](https://github.com/pinterest/gestalt/blob/master/packages/gestalt/src/Masonry/README.md#getpositions) the height of the item before laying it out on the grid. This is because that component is a generic one built for use cases beyond the homepage feed, where items can contain contents (e.g. photo captions) where the height of the items are not known beforehand. For a Pinterest homepage feed where the entire item is an image, we already know the intrinsic ratio of the image. With a fixed column width, we can calculate the required height of the image/item. Hence there's no need to render offscreen to measure.

Even though this layout approach's implementation is more complex, it is more flexible and allows for better accessibility as we can control the tab order of elements.

### Ordering items within columns
Now that we have established we should use absolute positioning for the items layout, the next thing we need to decide is how to order the items. Remember that pins that appear earlier in the feed should be placed above than the pins that appear later.

There are two common ways to order the pins:

- Round robin
- Height balancing

**Round robin placement**: In round robin placement, we put the pins in order into each column, wrapping around to the first column after the last, until all pins have been placed. In a three-column layout, pins are placed in these respective columns:

![96d1a42e2c5550cdbef9e3506364bcad.png](../_resources/96d1a42e2c5550cdbef9e3506364bcad.png)

The example below shows some pins that are ordered in a round robin fashion within a 3-column layout using absolute positioning, as well as the round robin algorithm to achieve it.

[Sandbox example](https://codesandbox.io/p/sandbox/pinterest-placement-round-robin-gnszl2?from-embed)

The benefit of this approach is that the algorithm runs in O(N) and is easy to implement (just modulo by the columns size).

However, this approach does not factor in the items height at all. As a result, some columns might end up taller than others. Taking the above example, the first column contains a few tall items and ends up being substantially taller than the second column. The result might not be visually pleasing.

**Height balancing**: In height balancing placement, the pins are placed within the shortest column, at the time of arrangement.

For a three-column layout, we can use an array of size 3 (`columnHeights`) where the value at the index represents the running total height of items in that column. For each pin, by looping through the `columnHeights` array once, we can determine the shortest column. Next, we calculate the pin's position based on the left-most column with the shortest current height and update that column's height for the newly-added pin.

```js
const pins = [
  { height: 160, id: 1 },
  { height: 70, id: 2 },
  { height: 130, id: 3 },
  { height: 160, id: 4 },
  // ...
];

const NUM_COLS = 3;
const GAP = 10;
const COL_WIDTH = 70;

function arrangeHeightBalanced(pins) {
  const columnHeights = Array(NUM_COLS).fill(0);

  // For each pin, augment with position data.
  return pins.map((pin) => {
    // Find the shortest column.
    let shortestCol = 0;
    for (let i = 1; i < NUM_COLS; i++) {
      if (columnHeights[i] < columnHeights[shortestCol]) {
        shortestCol = i;
      }
    }

    // Calculate the `left` value of the current pin.
    const left = shortestCol * COL_WIDTH + Math.max(shortestCol, 0) * GAP;
    // Calculate the `top` value of the current pin.
    const top = GAP + columnHeights[shortestCol];
    // Update the column height.
    columnHeights[shortestCol] = top + pin.height;

    return {
      ...pin,
      left,
      top,
      width: COL_WIDTH,
    };
  });
}

```

[Sandbox example](https://codesandbox.io/p/sandbox/pinterest-placement-balanced-tl4c5w?from-embed)

The result is a layout that the columns have generally balanced heights, which is more visually pleasing. Even though the algorithm now runs in O(N * columns), columns is a constant and at the scale of a realistic page containing hundreds of pins, the difference is negligible.

This `columnHeights` computation should be preserved within the Masonry component as state so that when the next page of pins is loaded and need to be arranged on the page, the column heights are immediately known, there's no need to tally the height of the columns again or arrange the pins that are already on-screen.

**Server computations**: Currently, the computations are done on the client. For Pinterest, although SSR is being used, the initial HTML for the pins does not contain positioning data and the positions are calculated on the client. This is probably because the server does not know the viewport dimensions of the client.

Pinterest grids have a few breakpoint widths for varying number of columns, so the total width of the container for each number of column is already known beforehand. It's technically possible to calculate the pins' positions on the server for all possible breakpoints and send them down to the client. The client just has to pick the pre-calculated values for the current breakpoint.

Assuming each pin is meant to be **200px wide** and have a **gap of 20px** between them, the total width of the container for each column:

![12dd3ca2a16dfd0b422cc716957a4190.png](../_resources/12dd3ca2a16dfd0b422cc716957a4190.png)

For laptops that have a display width of 1200px, there's only enough space for 5 columns, so they can directly use the pre-calculated pin positions for a 5-column layout. For mobile devices (usually under 600px), there's only enough space for 2 columns, so they can use the pre-calculated pin positions for a 2-column layout.

However, this is probably a micro-optimization and server computations are hard to reuse for the subsequent feed loads as the positioning of the next page rely on what is currently displayed on the clients. It is also cleaner from an engineering perspective to offload all the positioning computation work to clients.

### Responsiveness and resizing
By using absolute positioning, we're not leveraging much of the browser's layout capabilities and thus have to reimplement the position calculations on our own. As mentioned above, the position calculations done are only good for the current number of columns, if the number of available columns change (e.g. due to resizing), the position calculations have to be done again.

We can add a listener to the page for the `'resize'` event and recalculate the pin positions for the window width when the resize event is fired. However, the resize event fires very often, and recalculating so frequently can be expensive, especially if there are many pins on the page. Debouncing or throttling the event handler can reduce the number of times the positions are calculated. In reality, Pinterest uses debounce and only recalculate the layout when the window stops resizing.

One possible optimization is to calculate the positions for every possible number of columns while the browser is idle.

Another edge case to consider is when the scroll position is very far down the page and the user resizes from 5 columns to 2 columns, assuming the scroll position stays the same, the user will be seeing pins far higher up from before resizing and the user probably won't be able to resume from where they left off.

Ideally, the scroll position is adjusted so that the user is still viewing the same pins. We can remember the pins that are being shown before resizing, then calculate an average y-position of the pins post-resizing and set that as the scroll position, which is quite complicated to execute. In Pinterest's case, users are scrolled all the way to the top when the number of column changes. Resizing is not a common occurrence so it's probably not worth handling.

#### Pinterest's Masonry component
One awesome thing about Pinterest is that they have open sourced their React design system components, including the coveted Masonry component! There's even a [short writeup about how it works](https://github.com/pinterest/gestalt/blob/master/packages/gestalt/src/Masonry/README.md).

Some interesting things about the Masonry component:

- It accepts a list of items and does not do any data fetching. It fires a callback to inform the parent when the user scrolls past a given threshold based on the height of the container, to fetch the next page of items.
- Multiple layout strategies are supported:
	- **Default**: Yields a grid of items with a constant column width. This is the layout we have discussed above. 
	- **Uniform row**: Yields a grid of items in rows of uniform height. The height of the row will be the tallest item in the row. this is the same as a normal table behavior. 
	- **Full-width**: Yields a grid of items with flexible column widths. The grid will expand or shrink (by expanding or shrinking all column widths) to accommodate the width of its container; there are no breakpoints. 
- Virtualization can be enabled/disabled.

[Try the component here](https://codesandbox.io/p/sandbox/rhtyq2)

#### Boundary dimensions for images
Since the columns are fixed and the images try to preserve their original aspect ratios, some weird results can occur.

- **Very tall images**: Very tall images can take up the entire column, making the page look odd when one of the columns takes up the entire page.
- **Very wide images**: Very wide images will have a very small height as they have to maintain the aspect ratio within the column width. As a result, users might only see a thin horizontal line and appear barely visible. An image that that is 1000px wide and 20px tall will only have a height of 2px when put within a 100px-wide column while maintaining the aspect ratio.

Hence there should be a maximum and minimum heights for the images. For such images, they can be positioned using `object-fit: cover` so that the user can still get a glimpse some parts of the image.

### Advanced: paint scheduling
In the discussion above, we never really bothered with when the image is being painted to the screen. We assumed that when the `<img>` tag is added to the DOM, the image shows up instantaneously.

The term "painting" is often used in the context of web rendering engines and web browsers. It refers to the final step in the rendering process, where the browser takes the calculated layout and style information and creates the actual pixels that make up the visible content of the web page. This involves determining the position, size, and appearance of every element on the page, and then rendering them as pixels on the user's screen.

For users with a fast internet connection, images show up as soon as the `<img>` tags are added to the DOM. The images load very quickly and are painted to the screen almost immediately, within half a second. However, for users with slow internet connections, images take longer to load and take vastly different durations depending on the image size. This will result in images being painted in a random fashion which can look disorientating.

A more complex flow can be adopted to solve this problem:

1. **Load data**: Fetch feed data containing a list of pins from the server.
2. **Calculate layout**: Determine where to place a pin.
3. **Image loading:** The browser downloads the pin's image from the CDN. Images can be loaded without adding `<img>` tags to the DOM by doing `new Image()` in JavaScript and then setting the `src` field on the newly-created Image object.
4. **Paint scheduling**: Paints the image to the screen by adding an `<img>` tag to the DOM.


Steps 2 and 3 can be done in parallel. Step 4 requires step 2 and step 3 to be fully completed first. A naive and basic way is to wait for all the images to be loaded (Step 3 to be fully complete) before painting them to the screen.

#### Painting approaches
Overall, there are the following painting approaches:

1. **Simple default.** Render all the `<img>` tags and images will appear as soon as they're loaded. This is the case we covered at the start and provides a poor experience for users with slow internet connection.
2. **Load and paint sequentially.** This means loading a image, painting it, then repeat for the next image until there are no remaining images. This results in waterfall loading and painting, which is slow and makes little sense because image loading can be done in parallel across all the images. It's worse than the approach above.
3. **Load in parallel, paint all at once.** This is not ideal as users will have to wait for the slowest image to be loaded before they see any images at all.
4. **Load in parallel, paint in order.** This is the ideal approach where images are only shown if all images that come before it have been fully loaded.

To achieve the ideal approach, significant coordination is required and can be complex to implement, but thankfully the awesome React team has a solution to this. The `Suspense` component is able to coordinate rendering/painting and let us easily decide which parts of the UI should "pop in" together at the same time. There's also the `SuspenseList` component that allows revealing of items in order. `SuspenseList` is not yet released but you can see how it works from the [React 2019 keynote](https://www.youtube.com/watch?v=Tl0S7QkxFE4&t=921s) and [try out these examples](https://react-suspense-img.netlify.app/).

### Performance
#### List virtualization
Virtualization in the context of the DOM typically refers to a technique used in web development to improve the performance and efficiency of rendering large lists or collections of elements, such as those found in long tables, lists, or grids. Virtualization is employed to render only the elements that are currently visible in the viewport, rather than rendering the entire list. This technique helps optimize web page performance, especially when dealing with a large amount of dynamic content.

The basic idea of DOM virtualization is to:

1. Render only the elements that are within the visible portion of the web page.
2. Dynamically load or render additional elements as the user scrolls or interacts with the content.
3. Reuse and recycle DOM elements to minimize memory and performance overhead.

With an `absolutely`-positioned layout:

- The container knows exactly how tall it should be (the height of the tallest column) and can set its `height` style value.
- Offscreen DOM nodes that are meant to removed can be done so without affecting the position of the other items.
- The container has the `height` set, so removing items at the bottom do not cause the container to shrink, and there won't be scroll position changes.

On desktop, Pinterest allows a maximum number of 40 pins in the DOM at any one time. On mobile, it's around 10 - 20. It's possible to use dynamic values based on network conditions, device dimensions, device processing abilities, etc. 

Some observations:

1. The container has an initial height of 5386px but there are only 7 pins in the DOM (its direct children), which is roughly two screens worth of content.
2. As the user scrolls down, more pins are added to the bottom of the container element.
3. As the user scrolls further down, pins at the top are removed from the container element. There's a maxiumum of around 12 pins in the DOM at once.
4. The container height is increased to 10228px when the user has scrolled down far enough such that the next page of items have loaded.
5. Scrolling up causes the pins at the bottom to be removed from the DOM and pins higher up on the page are inserted back at the top.

### Reflows and repaints
Reflows and repaints have been mentioned earlier, but to recap the terms:

- **Reflow**: A browser reflow, often referred to as a layout or re-layout, is a process that web browsers go through to calculate the geometry and position of elements in a web page's Document Object Model (DOM) when certain changes are made to the page. These changes can include modifications to the content, structure, or style of the page, such as adding, removing, or altering elements, changing their dimensions, or adjusting their position.
- **Repaint**: A browser repaint, often referred to as "painting", is the process by which a web browser draws or renders the visible content of a web page on the user's screen. This involves creating and updating the pixels that make up the visual representation of the webpage, including text, images, backgrounds, and other graphical elements. A repaint operation is typically triggered when there are changes to the appearance of elements on a web page, such as updates to CSS styles, content changes, or user interactions that require redrawing parts of the page. Repaints occur after reflow, if a reflow was done.

Re-rendering a lot and quickly causes reflow and repainting which can result in browser lag. Since there are many images on the page, if images were painted one at a time to the screen, it would cause a significant reflow and repainting operations.

Techniques to reduce reflows and repaints include:

- **Batching DOM changes**: Make multiple changes to the DOM in a single operation, which can reduce the number of reflows triggered.
- **Using CSS transforms**: Applying transformations like translation, rotation, or scaling using CSS transforms generally doesn't trigger a reflow, making them a more efficient way to update the appearance of elements.
- **Avoiding forced synchronous layout**: Some DOM and style properties, when accessed, force the browser to perform a reflow. Avoiding these properties when not necessary can help reduce reflows.
- **Virtual scrolling and pagination**: For long lists of items, use virtual scrolling or pagination to limit the number of elements visible at once, reducing the impact of reflows.
- **Debouncing and throttling**: When responding to events that trigger reflows, use debouncing and throttling techniques to limit the frequency of these operations.

To reduce reflow and repaints, we can do the following:

- Paint images in order, aka only show an image if all that come before it have been fully loaded.
- Multiple images that load within a short period of time only cause one reflow and repaint. This feature is also built into [React `Suspense`](https://react.dev/reference/react/Suspense).
- Debounce/throttle masonry layout recalculation when the window resizes.
- List virtualization, so that the number of elements on the page is lower and fewer calculations are needed during reflows.

Some of them have already been mentioned above.

#### Automatic refresh when last fetched time was a while ago
If you leave your homepage tab open for a while and come back to it (maybe half an hour later), the site will blow away all the loaded items and refetch the entire feed. This serves the purpose of keeping the memory usage low as the loaded items can be purged from the client store. It's a good optimization because it's unlikely the user will be interested in stale pins that they have already scrolled past.

If there's no virtualization done, periodic refreshes can also clear out the DOM state, improving React reconciliation and lowering memory overhead.

#### Progressive Web App
Pinterest has made significant efforts to provide a Progressive Web App (PWA) experience for its users. A Progressive Web App is a web application that provides a native app-like experience in a web browser. PWAs aim to combine the best of web and mobile app experiences, offering features like offline access, push notifications, and fast loading times.

Their PWA efforts case study and retrospective have been published publicly:

- [A Pinterest Progressive Web App Performance Case Study](https://medium.com/dev-channel/a-pinterest-progressive-web-app-performance-case-study-3bd6ed2e6154)
- [A one year PWA retrospective](https://medium.com/pinterest-engineering/a-one-year-pwa-retrospective-f4a2f4129e05)

### Network
Since there are many images to be shown on screen, there are many images to be downloaded, the browser makes multiple simultaneous network requests. Traditionally, browsers have a restriction on the maximum number of parallel HTTP connections per domain. This resulted in web services using different domain names for hosting their images. However, HTTP/2 allows multiple requests over a single connection, reducing the need for multiple parallel connections.

Pinterest uses a single domain for their CDN (`pinimg.com`) and supports HTTP/3, so the page doesn't run into the issue of exceeding the maximum number of parallel connections.

### User experience
#### 	Loading states
For devices on slow networks, images might take a while to load so showing a placeholder for the image improves the user experience. Instead of showing a generic gray-colored box as a placeholder, Pinterest has identified dominant colors for each image and renders a box with that dominant color as the background.

![ed13e27a6ccea26919a28615417529db.png](../_resources/ed13e27a6ccea26919a28615417529db.png)

### Error handling
Images might sometimes fail to load, and there are a few ways to handle them:

- **Ignore the failed images**. This is possibly the easiest and reasonable way. Pinterest feeds are somewhat random and users will not realize if a particular pin has failed to load.
- **Retry loading and insert later**. The client can retry loading the image and if it's successful, insert it at the bottom of the current feed or as part of the next page's rendering.
- **Show error message**. An error message can be shown within the allocated space for that pin. However, there might not be enough space to display the message, especially if the message is long (possible in certain languages) and the available pin space is small.

### Internationalization (i18n)
There isn't too much to discuss in terms of i18n for this question because the focus ins on the layout. For RTL languages, the masonry layout algorithm can be easily tweaked to start from the right instead of the left.

### Accessibility (a11y)

#### Screen readers
- `alt` attribute for `<img>`s.
- `role="list"` attribute for feed container.
- `role="listitem"` attribute for feed items.

#### Keyboard support
Ensure tabbing order matches browsing order, which can be done with absolute positioned pin.
When the user is far down on the page, pressing `Tab` should focus on a pin within the viewport than the pin at the top of the feed.

### Other interactions
The focus of this question is on the feed masonry layout and we have intentionally omitted details about other common feed interactions:

- **Viewing details of a pin.** This show the pin details within a modal, displaying the image and other details like title, description, author. Dismissing the modal will bring the user back to the feed with the scroll position restored, allowing them to continue browsing from where they left off.
- **Saving a pin.** Leverage optimistic updates.
- **Pin creation.** Uploading images and filling in essential information like title and description.
- **Extra actions only revealed upon interaction.** Lazy load the code for these non-essential actions only when they're being used or when the page is idle.

## References

- [A Pinterest Progressive Web App Performance Case Study](https://medium.com/dev-channel/a-pinterest-progressive-web-app-performance-case-study-3bd6ed2e6154)
- [A one year PWA retrospective](https://medium.com/pinterest-engineering/a-one-year-pwa-retrospective-f4a2f4129e05)
- [Improving GIF performance on Pinterest](https://medium.com/pinterest-engineering/improving-gif-performance-on-pinterest-8dad74bf92f1)
- Gestalt (Pinterest's design system)
	- [Masonry component](https://gestalt.pinterest.systems/web/masonry)
	- [How masonry works](https://github.com/pinterest/gestalt/blob/master/packages/gestalt/src/Masonry/README.md)





