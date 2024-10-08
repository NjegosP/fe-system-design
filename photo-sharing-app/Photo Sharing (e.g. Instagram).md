# Photo Sharing (e.g. Instagram)

## Question
Design a photo sharing application that contains a list of photo posts created by users. Users can create new posts containing photos.

**Note**: This application shares many similarities with the News Feed system design application and the image carousels within the feed post will benefit from the Image Carousel system design. Please have a read of that question before starting on this. For this question, we will focus on things that haven't been covered, the discussion will be centered around the photos and images.

## Requirements exploration
### What are the core features to be supported?
Browse a feed containing image and video posts by people a user follows.
Upload photos, add captions and apply filters to them before posting.
### What pagination UX should be used for the feed?
Infinite scrolling, meaning more posts will be added when the user reaches the end of their feed.

### Is the user able to add captions to individual photos/images or only one overall caption to the post?
Only one overall caption to the post.

### What devices will the application be used on?
Primarily mobile, but should be usable on desktop and tablet as well.

## Architecture / high-level design
### Rendering approach
Photo sharing applications have the following characteristics:

- Posts are viewable by non-logged in users.
- Posts are searchable via search engines.
- Application is interaction heavy due to post composing and liking/commenting functionality on posts.
- Fast initial loading speed is desired.

These are also characteristics of a News Feed system design where there's a good amount of static content which also require interaction. Hence a combination of server-side rendering with hydration and subsequent client-side rendering would be ideal. In reality, instagram.com is built using the same technology stack as facebook.com which uses React server-side rendering with hydration.

### Architecture diagram

![592b48dae487324eced0e68867138228.png](../_resources/592b48dae487324eced0e68867138228.png)

- **Server**: Provides HTTP APIs to fetch feed posts, upload images, and to create new image posts.
- **Controller**: Controls the flow of data within the application and makes network requests to the server.
- **Client store**: Stores data needed across the whole application. In the context of a photo sharing application, most of the data in the store will be server-originated data needed by the feed UI.
- **Feed UI**: Contains a list of image posts and the UI for creating new posts.
	- **Image post**: A post containing a list of one to many images.
	- **Post composer**: UI for uploading images, applying filters to them and adding a caption before submitting to the server.

## Data model
A photo feed shows a list of image posts fetched from the server, hence most of the data involved in the application will be server-originated data. The only client-side data needed is for user-uploaded images and user-written captions in the post composer.

![d51dd4101527a6a41047f022165b07f8.png](../_resources/d51dd4101527a6a41047f022165b07f8.png)

## Interface definition (API)
### Feed list API
Like News Feed's list API, the photo sharing application's feed list API should also use a cursor-based pagination approach.

### Post creation API
All posts in a photo sharing application has attached images. Hence the post creation process in such apps is more complex.

Let's break down the post creation into steps:

1. User selects photos from their device.
2. User is able to perform light edits on their photos: resize/crop/filter.
3. User adds a caption to the post.
4. User submits post.

There are two common ways to implement post creation functionality involving images:

1. Create a single API for both image uploading and post creation.
2. Create separate APIs for image uploading and post creation.

**1. Create a single API for both image uploading and post creation**

![e82f801501cb8c28f057bed463ac12ed.png](../_resources/e82f801501cb8c28f057bed463ac12ed.png)

Pros:

- Simple to implement, upload all required data in one request.

Cons:

- API logic is more complex because it has to do multiple things on the back end (though that's not really a concern of the front end).
- Upload will take longer because the post images have to be uploaded within one request.
- API has to take in both text-based data and media data.

**2. Create separate APIs for image uploading and post creation**

![bfa410f133480f2fd94fb8a8cd59c364.png](../_resources/bfa410f133480f2fd94fb8a8cd59c364.png)

Pros:

- Image upload stage can be done in a an async fashion. Images can be uploaded after the image editing stage in the background while the user is drafting the post caption.
- Images can be uploaded in parallel across multiple HTTP requests which is faster than uploading all the images in a single HTTP request which will take longer.
- Can leverage existing general image uploading APIs, if any.

Cons:

- More coordination needed in the client. The client needs to wait for all the images to be uploaded and get a reference to the uploaded images and use them when creating the post.

Comparing the two approaches, approach #2 of separate APIs would be better. The image uploading API can be a general one that is used across the entire website and not just for the post creation flow.

## Optimizations and deep dive
### Feed optimizations
These optimizations done in News Feed system design also apply to the feed of photo sharing applications.

- Rendering approach
- Infinite scrolling implementation
- Virtualized lists
- Code splitting JavaScript
- Loading indicators
- Preserving feed scroll position
- Lazy load code
- Optimistic updates
- Timestamp rendering
- Icon rendering

### Image carousel optimizations
The images within a post are shown as a carousel, so the Image Carousel system design article is also useful for this question and the same optimizations apply.

### Rendering images
- Use modern image formats such as `WebP` which provides superior lossless and lossy image compression.
- `<img>`s should use proper `alt` text.
	- Instagram allows users to provide `alt` text for each image. If the user doesn't specify the `alt` text, Machine Learning and Computer Vision techniques can be used to process the images and generate a description.
- Image loading based on device screen properties
	- Send the browser dimensions in the initial request (or subsequent is also fine) and server can decide what image size to return.
	- Use `srcset` if there are image processing (resizing) capabilities to load the most suitable image file for the current viewport.
- Adaptive image loading based on network speed
	- **Devices with good internet connectivity/on WiFi**: Prefetch offscreen images that are not in the viewport yet but about to enter viewport.
		- **Poor internet connection**: Render a low-resolution placeholder image and require users to explicitly click on them to load the high-resolution image.

## Image editing
### Cropping and resizing
Cropping and resizing of the images can be done via HTML5' `<canvas>`.

```js
const canvas = document.getElementById('image-editor');
const context = canvas.getContext('2d');
const image = new Image();
image.src = 'https://greatimages.com/example.jpg';
context.drawImage(image /* Other parameters */);
```

Instagram website allows editing by simulating the result with inline styles to modify the transform styling and positioning of the image.

### Filters
CSS provides a filter property which allows applying `filter` functions like `blur`, `contrast`, `hue`, `sepia` to name a few. By using a combination of these filter functions, you can achieve Instagram-like filtering functionalities right within your browser.

Here are examples of filter functions to achieve the Clarendon and Gingham filter effects from Instagram, taken from the awesome [Instagram.css](https://picturepan2.github.io/instagram.css/).

```css
.filter-1977 {
  filter: sepia(0.5) hue-rotate(-30deg) saturate(1.4);
}

.filter-brannan {
  filter: sepia(0.4) contrast(1.25) brightness(1.1) saturate(0.9) hue-rotate(-2deg);
}
```

The following images are enhanced right within your browser using these CSS filters.

![2c1f037772ca9e4243ed501c73e18ead.png](../_resources/2c1f037772ca9e4243ed501c73e18ead.png)

Ultimately, the final image still has to be converted into an image blob before sending to the server and this conversion can be done with libraries like [html2canvas](https://github.com/niklasvh/html2canvas).

## Accessibility (a11y)
### Screen readers
- `<img>` tags should have a meaningful alt description specified or use an empty string.
- Add `aria-labels` to the prev/next buttons within an image carousel.
- Icon-only buttons should have the appropriate `aria-labels` if there are no accompanying labels (e.g. heart, share, bookmark).
### Keyboard support
- Use the `<button>` HTML tag for the Prev/Next buttons where possible so that the buttons are focusable.
- Add `<div role="region" aria-label="Image Carousel" tabindex="0">` to make the carousel focusable and attach Left/Right keydown handlers to allow scrolling through the images with the keyboard.

## References
- [Making Instagram.com faster: Part 1](https://instagram-engineering.com/making-instagram-com-faster-part-1-62cc0c327538)
- [Making Instagram.com faster: Part 2](https://instagram-engineering.com/making-instagram-com-faster-part-2-f350c8fba0d4)
- [Making Instagram.com faster: Part 3 — cache first](https://instagram-engineering.com/making-instagram-com-faster-part-3-cache-first-6f3f130b9669)
- [Making instagram.com faster: Code size and execution optimizations (Part 4)](https://instagram-engineering.com/making-instagram-com-faster-code-size-and-execution-optimizations-part-4-57668be796a8)
- [Launching Instagram Messaging on desktop](https://about.instagram.com/blog/engineering/launching-instagram-messaging-on-desktop)
- [Crafting an Accessible Instagram Feed](https://about.instagram.com/blog/engineering/crafting-an-accessible-instagram-feed)