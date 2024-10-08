# E-commerce Marketplace (e.g. Amazon)

## Question
Design an e-commerce website that allows users to browse products and purchase them.

## Requirements exploration
### What are the core features to be supported?
- Browsing products.
- Adding products to cart.
- Checking out successfully.
### What are the pages in the website?
- Product listing page (PLP)
- Product details page (PDP)
- Cart page
- Checkout page
### What product details will be shown on the PLPs and PDPs?
- PLPs: Product name, Product image, Price.
- PDPs: Product name, Product images (multiple), Product description, Price.
### What does the user demographics look like?
International users of a wide age range: US, Asia, Europe, etc.

### What are the non-functional requirements?
Each page should load under 2 seconds. Interactions with page elements should respond quickly.

### What devices will the application be used on?
All possible devices: laptop, tablets, mobile, etc.

### Do users have to be signed in to purchase?
Users can purchase as a guest without being signed in.

## Architecture / high-level design

![ee0743354621c1323514cd55ab64654a.png](../_resources/ee0743354621c1323514cd55ab64654a.png)

### Component responsibilities
- **Server**: Provides HTTP APIs to fetch products data, cart items, modify the carts and create orders.
- **Controller**: Controls the flow of data within the application and makes network requests to the server.
- **Client Store**: Stores data needed across the whole application. Since there are many pages with data some amount of overlapping data, a client store is useful for sharing data across sections of a page and across pages.
- **Pages**:
	- **Product List**: Displays a list of product items that can be added to the cart.
	- **Product Details**: Displays details of a single product along with additional details.
	- **Cart**: Displays added cart items and allows changing of quantity and deleting added items.
	- **Checkout**: Displays address and payment forms the user has to complete in order to place the order.

### Server-side rendering or client-side rendering?
Firstly let's understand what the two terms mean:

- **Server-side rendering (SSR)**: The traditional way of building websites where the server fetches all necessary data, uses them to create the final markup and sends down the HTML needed every time a user visits a page. Most of the rendering work is done on the server.
- **Client-side rendering (CSR)**: The server sends down initial HTML which contains the JavaScript to bootstrap the application. The client then fetches the necessary data, combines it with templates and creates the final page, all within the browser. CSR is typically used with Single-page Application models and subsequent navigation don't require a full page refresh. Most of the rendering work is done on the client.

Benefits of SSR:

- Performance is generally better, First Contentful Paint score is high and SSR-ed pages appear faster than CSR.
- Lower Cumulative Layout Shift score as the final HTML is already present.
- SSR allows for personalization of pages (user-specific content) as opposed to static site generation. Personalization is an important factor for conversion as e-commerce platforms scale up.

Downsides of SSR:

- Page transitions are slower because the entire page has to be constructed on the server for every request.

SEO is important for e-commerce websites, hence SSR should be a priority.

### Single-page application (SPA) or Multi-page application (MPA)?
SPAs by default use CSR and MPAs by default use SSR. While CSR and SSR are two ends of the rendering spectrum, somewhere in the middle lies a hybrid mode called universal rendering (or SSR with hydration) where the server renders the full HTML but after that, rendering and navigation becomes client-side.

Most universal rendered-sites use popular UI frameworks (e.g. React, Vue, and Angular) and the page will need to be hydrated (add event handlers) after initial load. Hydration also brings about the [double data problem](https://web.dev/articles/rendering-on-the-web#a-rehydration-problem-one-app-for-the-price-of-two).

Which application architecture should be used? The most important factor is that SSR is being used, whether SPA or MPA doesn't matter as much. As seen above, both are viable, as long as your website has good performance.

Real world technologies to implement the following rendering and architecture choices:

![fd25d9f92f106a0a05b0895c63a83c8d.png](../_resources/fd25d9f92f106a0a05b0895c63a83c8d.png)

### What top e-commerce sites use
Let's take a look at e-commerce sites in the wild and their rendering choices:

![6bb85ec470b23fe5ab60e2ec03576bca.png](../_resources/6bb85ec470b23fe5ab60e2ec03576bca.png)

**All these e-commerce sites use SSR!** This suggests the importance of using SSR for e-commerce sites.

The discussion below assumes that we'll be using a universal rendering approach of SSR + SPA.

## Data model
There are quite a number of entities involved in an e-commerce website due to the complexity of the user flow spanning across multiple pages.

![598d5b4fe9c4a93cf64a55e85a37d617.png](../_resources/598d5b4fe9c4a93cf64a55e85a37d617.png)

Few points for discussion:

- The `Cart` entity belongs to the Client Store because some websites might want to show the number of cart items in the navbar or have a popup to allow users to quickly access the cart items and make modifications. If there's no such need then it's acceptable for the cart to belong to the Cart page.
- `Cart` and `CartItem` have `total_price` and price fields respectively fetched as part of the server response instead of having the client compute the price (multiply the quantity by the `unit_price`) to give the flexibility of applying discounts due to bulk purchase or use of promo codes. The price computation logic is defined on the server, so the final price should be computed on the server and the client should rely on the server to calculate the total price instead of making its own calculations.
- Since our website has an international audience, we should have localized prices in the user's currency, hence the `currency` field.

## Interface definition (API)
We need the following HTTP APIs:

Product Information
1. Fetch products listing
	- Fetch a particular product's detail
	- Cart Modification
2. Add a product to the cart
	- Change quantity of a product in the cart
	- Remove a product from the cart
3. Complete the order

We will omit discussing about the APIs between the client components because the data format and functionalities are similar to the HTTP APIs.

We can also assume that a user only has a maximum of one cart and the user's current cart can be retrieved on the server. Hence we can omit passing the cart ID as arguments for any APIs related to cart modification.

### Fetch products listing

![723c2ce9d8adcdffdd7e33d3d0e4d927.png](../_resources/723c2ce9d8adcdffdd7e33d3d0e4d927.png)

#### Sample response

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
      "id": 123, // Product ID.
      "name": "Cotton T-shirt",
      "primary_image": "https://www.greatcdn.com/img/t-shirt.jpg",
      "unit_price": 12,
      "currency": "USD"
    }
    // ... More products.
  ]
}
```

We use offset-based pagination here as opposed to cursor-based pagination because:

1. Having page numbers is useful for navigating between search results and jumping to specific pages.
2. Product results do not suffer from the stale results issue that much as new products are not added that quickly/frequently.
3. It's useful to know how many total results there are.
For a more in-depth comparison between offset-based pagination and cursor-based pagination, refer to the News Feed system design article.

### Fetch product details

![ac88eb966bc33d76ac8f26ec6b390958.png](../_resources/ac88eb966bc33d76ac8f26ec6b390958.png)

#### Sample response

```js
{
  "id": 123, // Product ID.
  "name": "Cotton T-shirt",
  "primary_image": "https://www.greatcdn.com/img/t-shirt.jpg",
  "image_urls": [
    "https://www.greatcdn.com/img/t-shirt.jpg",
    "https://www.greatcdn.com/img/t-shirt-black.jpg",
    "https://www.greatcdn.com/img/t-shirt-red.jpg"
  ],
  "unit_price": 12,
  "currency": "USD"
}
```

### Add a product to cart

![3f9d656e0bb1f1d459b0721f07b065df.png](../_resources/3f9d656e0bb1f1d459b0721f07b065df.png)

#### Sample response
The updated cart object is returned.

```js
{
  "id": 789, // Cart ID.
  "total_price": 24,
  "currency": "USD",
  "items": [
    {
      "quantity": 2,
      "price": 24,
      "currency": "USD",
      "product": {
        "id": 123, // Product ID.
        "name": "Cotton T-shirt",
        "primary_image": "https://www.greatcdn.com/img/t-shirt.jpg"
      }
    }
  ]
}
```

### Change quantity of product in cart

![981da2b7bddc6f76d447b3e17f342ff0.png](../_resources/981da2b7bddc6f76d447b3e17f342ff0.png)

#### Sample response
The updated cart object is returned.

```js
{
  "id": 789, // Cart ID.
  "total_price": 24,
  "currency": "USD",
  "items": [
    {
      "quantity": 3,
      "price": 36,
      "currency": "USD",
      "product": {
        "id": 123, // Product ID.
        "name": "Cotton T-shirt",
        "primary_image": "https://www.greatcdn.com/img/t-shirt.jpg"
      }
    }
  ]
}
```

### Remove product from cart

![958c0a3dac8346ba7db2ae8378684a8a.png](../_resources/958c0a3dac8346ba7db2ae8378684a8a.png)

#### Sample response
The updated cart object is returned.

```js
{
  "id": 789, // Cart ID.
  "total_price": 0,
  "currency": "USD",
  "items": []
}
```

### Place order

![8fde61aaab43df60b9cc0052c4c2d590.png](../_resources/8fde61aaab43df60b9cc0052c4c2d590.png)

#### Sample response
The order object is returned upon successful order creation.

```js
{
  "id": 456, // Order ID.
  "total_price": 36,
  "currency": "USD",
  "items": [
    // ... Same items as per the cart.
  ],
  "address_details": {
    "name": "John Doe",
    "country": "US",
    "address": "1600 Market Street",
    "city": "San Francisco"
    // ... Other address fields.
  },
  "payment_details": {
    // Only show the last 4 digits.
    // We shouldn't be storing the credit card number
    // unencrypted anyway.
    "card_last_four_digits": "1234"
  }
}
```

### Notes
- Depending on whether we want to optimize for returning users, we might want to save the address and payment details on the cart object so that people who abandoned the cart after filling up the checkout form but before placing the order and resume from where they left off without having to fill up the forms again.

## Optimizations and deep dive
### Performance
Performance is absolutely critical for e-commerce websites. Seemingly small performance improvements can lead to significant revenue and conversion increases. A study by [Google and Deloitte](https://web.dev/case-studies/milliseconds-make-millions) showed that even a 0.1 second improvement in load times can improve conversion rates across the purchase funnel. [web.dev by Google has a long list of case studies of how improving site performance led to improved conversions](https://web.dev/case-studies).

### General performance tips
- Code split JavaScript by routes/pages.
- Split content into separate sections and prioritize above-the-fold content while lazy loading below-the-fold content.
- Defer loading of non-critical JavaScript (e.g. code needed to show modals, dialogs, etc.).
- Prefetch JavaScript and data needed for the next page upon hover of links/buttons.
	- Prefetch full product details needed by PDPs when users hover over items in PLPs.
	- Prefetch checkout page while on the cart page.
- Optimize images with lazy loading and adaptive loading.
- Prefetching top search results.

### Core Web Vitals
Know the various core web vital metrics, what they are, and how to improve them.

- [Largest Contentful Paint (LCP)](https://web.dev/articles/lcp): The render time of the largest image or text block visible within the viewport, relative to when the page first started loading.
	- Optimize performance – loading of JavaScript, CSS, images, fonts, etc.
- [First Input Delay (FID)](https://web.dev/articles/fid): Measures load responsiveness because it quantifies the experience users feel when trying to interact with unresponsive pages. A low FID helps ensure that the page is usable.
	- Reduce the amount of JavaScript needed to be executed on page load.
- [Cumulative Layout Shift (CLS)](https://web.dev/articles/cls): Measures visual stability because it helps quantify how often users experience unexpected layout shifts. A low CLS helps ensure that the page is delightful.
	- Include size attributes on images and video elements or reserve space for these elements using CSS `aspect-ratio` to reserve the required space for images while the images are loading. Use CSS `min-height` to minimize layout shifts while elements are lazy loaded.

### Search engine optimization
SEO is extremely important for e-commerce websites as organic search is the primary way people discover products.

- PDPs should have proper `<title>` and `<meta>` tags for description, keywords, and [open graph tags](https://ahrefs.com/blog/open-graph-meta-tags/).
- Generate a `sitemap.xml` to tell crawlers the available pages of the website.
- Use [JSON structured data](https://developer.chrome.com/docs/lighthouse/seo/structured-data) to help search engines understand the kind of content on your page. For the e-commerce case, the [`Product` type](https://developers.google.com/search/docs/appearance/structured-data/product) would be the most relevant.
- Use semantic markup for elements on the page, which also helps accessibility.
- Ensure fast loading times to help the website rank better in Google search.
- Use SSR for better SEO because search engines can index the content without having to wait for the page to be rendered (in the case of CSR).
- Pre-generate pages for popular searches or lists.

The Travel Booking (Airbnb) system design article goes into more details about SEO.

### Images
Images are one of the largest contributors to page size and serving optimized images is absolutely essential on e-commerce websites which are image heavy where every product has at least one image.

- Use the WebP image format which is the most efficient image format that currently exists. eBay uses WebP format across all their web, Android and iOS apps. Ensure you are able to articulate on a high level why WebP format is superior.
- Images should be hosted on a CDN.
- Define the priority of the images and divide them into critical and non-critical assets.
- Lazy load below-the-fold images.
	- Use `<img loading="lazy">` for non-critical images.
- Load critical images early.
	- Inline the image within the HTML as a data blob so that there's no need to make a separate HTTP request to fetch the image.
	- Using `<link rel="preload">` so they download as soon as possible.
- [Adaptive loading](https://web.dev/articles/adaptive-loading-cds-2019) of images, loading high-quality images for devices on fast networks and using lower-quality images for devices on slow networks.

### Form optimizations
Filling in forms is a huge part of the checkout flow and a very troublesome one at that. It is the last step of the checkout process and nailing a good checkout experience will greatly help in the conversion rate.

Completing forms is especially painful for mobile devices and extra attention has to be given to optimize forms for mobile. There are two kinds of forms to fill out during checkout, which are the shipping address forms and credit card forms.

### Country-specific address forms
Different countries have different address formats. To optimize for global shipping, having localized address forms help greatly in improving conversions and that users do not drop off when filling out the address forms because they do not know how to understand certain fields. For example:

- "ZIP Code"s are called "Postal Code"s in the United Kingdom.
- There are no states in the Japan, only prefectures.
- Different countries have their own postal/zip code formats and require different validation.

It is a hassle to have to find out these country-specific knowledge and also build these yourself, this is where services like [Stripe Checkout](https://checkout.stripe.dev/preview) come in helpful by providing a localized checkout form. Users will complete the rest of the payment flow on Stripe's platform.

### Examples of Stripe checkout form for different countries

![287b0fafe1e18f4f6895167abfaf7967.png](../_resources/287b0fafe1e18f4f6895167abfaf7967.png)

*Further reading: [Frank's Compulsive Guide to Postal Addresses](https://www.columbia.edu/~fdc/postal/) provides useful links and extensive guidance for address formats in over 200 countries.*

### Optimize autofilling
Filling up forms, especially long ones, are prone to typo errors. Most modern browsers have a feature called autofill, where they help users enter data faster and avoid filling up the same form data again by using values from similar forms filled previously.

Help users autofill their address forms by specifying the right `type` and `autocomplete` values for the form `<input>`s for the shipping address forms and credit card forms.

### Shipping Address Form

![cd149b2f4327b3d9c837f4d4f4ca3950.png](../_resources/cd149b2f4327b3d9c837f4d4f4ca3950.png)

### Credit Card Form

![9883d0541a83ac7925691240e532f72c.png](../_resources/9883d0541a83ac7925691240e532f72c.png)

### Notes

- `inputmode="numeric"` provides a hint to browsers for devices with onscreen keyboards to help them decide which keyboard to display. Unlike `<input type="number">`, `inputmode="numeric"` does not prevent users from typing non-numeric values, they simply affect the keyboard being shown. As these numeric fields are not related to quantity, the chevrons which appear when using `<input type="number">` are not too helpful. `<input type="text" inputmode="numeric">` also works with the `maxlength`/`minlength`/`pattern` attributes, but do not work with `<input type="number>`.

Read more about forms:

- [Autofilling | web.dev](https://web.dev/learn/forms/autofill/)
- [Autofill on Browsers: A Deep Dive | eBay Engineering Blog.](https://innovation.ebayinc.com/tech/engineering/autofill-deep-dive/)

### Alternative ways of address input
Instead of making users fill up a form containing granular address fields, there are other approaches which may be easier, at the cost of engineering complexity or reliance on external services:

1. **Address search/autocomplete**: Allow users to search for an address by typing in their street number and letting them select from a list of suggestions. This reduces typos and is generally faster. However, users should still be given the option to override some values in case none of the suggestions are correct, which can happen due to an outdated address database. Google Maps JavaScript API provides this via [Place Autocomplete library](https://developers.google.com/maps/documentation/javascript/place-autocomplete).

2. **Selecting address location from a map**: Open up a map and allow users to pinpoint a location on the map. This is less common for checkout addresses and more common for ridehailing applications.

### Error messages
Leverage client-side validation and clearly communicate any form errors. Connect the error message to the `<input>` via `aria-describedby` and use `aria-live="assertive"` for the error message.

```html
<form>
  <div>
    <label for="name">Name</label>
    <input
      required
      minlength="6"
      type="text"
      id="name"
      name="name"
      aria-describedby="name-error-message" />
    <span
      id="name-error-message"
      aria-live="assertive"
      class="name-error-message">
      Name must have at least 6 characters!
    </span>
  </div>
  <button>Submit</button>
</form>
```

*Source: [Help users find the error message for a form control | web.dev](https://web.dev/learn/forms/accessibility/#help-users-find-the-error-message-for-a-form-control)*

### Focus states
Make the currently focused form control visually different from the other form inputs to help users identify which element is being focused.

### Best practices for payment and address forms
Read more about building good [Payment forms](https://web.dev/learn/forms/payment/) and [Address forms](https://web.dev/learn/forms/address/) and [general form best practices on web.dev](https://web.dev/articles/payment-and-address-form-best-practices).

### Internationalization (i18n)
- Have pages translated in the supported languages.
	- Set the `lang` attribute on the `html` tag (e.g. `<html lang="zh-cn">`) to tell browsers and search engines the language of the page which helps browsers offer a translation of the page.
	- Provide support for RTL languages by using [CSS Logical Properties](https://web.dev/learn/css/logical-properties/)
- Forms
	- Use a single form fields for names.
	- Allow for various address formats. This is covered in the section above on address forms.

Read more:

- [Internationalization and localization | Forms | web.dev](https://web.dev/learn/forms/internationalization/)
- [Internationalization | Design | web.dev](https://web.dev/learn/design/internationalization/)

### Accessibility
- Use semantic elements where possible: headings, buttons, links, inputs instead of styled `<div>`s.
- `<img>` tags should have the `alt` attribute specified or left empty if the merchant did not provide a description for them.
- Building accessible forms has been covered in detail above. In summary:
	- `<input>`s should have associated `<label>`s.
	- `<input>`s are linked to their error messages via `aria-describedby` and error messages are announced with `aria-live="assertive"`.
	- Use `<input>`s of the correct types and appropriate validation-related attributes like `pattern`, `minlength`, `maxlength`.
	- Visual order matches DOM order.
	- Make the currently focused form control obvious.

*Reference: [Accessibility | Forms | web.dev](https://web.dev/learn/forms/accessibility/) and [WebAIM: Creating Accessible Forms](https://webaim.org/techniques/forms/)*

### Security
Since payment details are highly sensitive, we have to make sure the website is secure:

- Use HTTPS so that all communication with the server is encrypted and that other users on the same Wi-FI network cannot intercept and obtain any sensitive details.
- Payment details submission API should not be using HTTP `GET` because the sensitive details will be included as a query string in the request URL which will get added to the browsing history which is potentially unsafe if the browser is shared by other users. Use HTTP `POST` or `PUT` instead.

*Source: [Security and privacy | web.dev](https://web.dev/learn/forms/security-privacy/)*

### User experience
- Make the checkout page clean (e.g. minimal navbar and footer) and remove distractions to reduce bounce rate.
- Allow persisting cart contents (either in database or cookies) as some people spend time researching and considering, only making the purchase during subsequent sessions. You don't want them to have to add all of the items to their cart again.
- Make promo code fields less prominent so that people without promo codes will not leave the page to search the web for a promo code. Those who have a promo code beforehand will take the effort to find the promo code input field.

## References
- [Shopping for speed on eBay.com | web.dev](https://web.dev/case-studies/shopping-for-speed-on-ebay)
- [Case Study | web.dev](https://web.dev/case-studies)
- [How Rakuten 24's investment in Core Web Vitals increased revenue per visitor by 53.37% and conversion rate by 33.13% | web.dev](https://web.dev/case-studies/rakuten)
- [Speed By A Thousand Cuts](https://innovation.ebayinc.com/tech/engineering/speed-by-a-thousand-cuts/)
- [How focusing on web performance improved Tokopedia's click-through rate by 35% | web.dev](https://web.dev/case-studies/tokopedia)
- [Autofill on Browsers: A Deep Dive](https://innovation.ebayinc.com/tech/engineering/autofill-deep-dive/)
- [Rendering on the Web | web.dev](https://web.dev/articles/rendering-on-the-web)







	







