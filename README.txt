## Bypassing FlySafair's Queue-It Implementation

**Disclosure:** I reached out to FlySafair on the day of the sale. I am yet to hear back from them.

---

Every year FlySafair runs a birthday sale on flights, selling tickets at a price matching their current age  this year that was R12 a flight. Naturally, this draws enormous excitement and traffic, so FlySafair uses a virtual queue system called [Queue-It](https://queue-it.com) to manage the load. The idea is straightforward: every visitor is assigned a unique ID, placed in a queue, and randomly selected when it's their turn to purchase.

Having been through this before, I knew the queue would be back this year. So I decided to get ahead of it.

---

## The Setup: Mapping the API

Before the sale went live I spent some time extracting every API endpoint the FlySafair website uses to complete a booking. This worked well in testing  I had a clean picture of the entire booking flow. Then sale day arrived, and FlySafair had added a twist.

Each API endpoint now validated a custom header. Something like:

```
X-FSF-Token: HGHQWE1SDWA
```

The name and value were effectively random strings, but crucially the value was **static**  the same for every request during the sale. The check was only enabled during the sale window, and the only way to legitimately obtain the header was to load the FlySafair site itself, since the JS bundle that makes those API calls has to carry the value.

Which brought me back to the site. And the queue.

---

## The Queue: What Actually Happens

Hitting `https://www.flysafair.co.za/` during the sale produces this:

```http
HTTP/2 302 Found
Location: https://flysafair.queue-it.net/?c=flysafair&e=r12bdaybackup&...
X-Queueit-Connector: cloudflare
Server: cloudflare
```

The `X-Queueit-Connector: cloudflare` header tells the story. The queue check is implemented as a **Cloudflare Worker** sitting at the edge before the request ever reaches FlySafair's origin server. If the Worker doesn't find a valid Queue-It token in your cookies, it issues a 302 straight to the queue page. You aren't getting through without waiting.

So the question became: is the Worker applied to every path on the site, or just some of them?

---

## The Sitemap Scan

Before the sale I downloaded FlySafair's public sitemap the XML file they publish specifically so search engines can find all their pages. I wrote a small Python scanner that probed each URL in the sitemap and logged the response signature. The goal was simple: find any URL that returns the real site rather than a redirect to the queue.
Most pages fell into one of two categories. Some returned the expected `302` to queue-it. Others returned a `200 OK` but with an HTML blob that eventually redirected the browser to the queue client-side still gated.

Two paths stood out immediately:

```
/check-in
/manage
```

Their responses looked like this:

```http
HTTP/2 200 OK
Via: 1.1 eaecd2156c5a302266e719d4c9e21862.cloudfront.net (CloudFront)
X-Amz-Server-Side-Encryption: AES256
X-Amz-Cf-Pop: JNB51-P2
Content-Type: text/html; charset=utf-8
```

No `X-Queueit-Connector`. No redirect. Coming from **AWS CloudFront** rather than Cloudflare. These routes are served from a completely separate origin an S3 bucket fronted by CloudFront  that was never put behind the Queue-It Worker in the first place. The Cloudflare Worker simply isn't in the request path for these URLs.

And critically, what they serve isn't a lightweight check-in stub. They serve the **entire SPA bundle**  the same `/static/js/app.<hash>.js` that powers every other page on the site, including the booking flow and the API calls with the secret header baked in.

---

## The Client-Side Queue Check

At this point it looked like simply pointing a browser at `/check-in` should be enough. Not quite.

Somewhere in the JS bundle, Queue-It's client-side connector runs on page load. It phones home to `assets.queue-it.net`, validates whether the current visitor has a queue token, and if not, redirects to the queue  same outcome as the server-side check, just happening after the page has loaded rather than before.

The good news is that this check fires **after** the browser has already received the complete SPA bundle. That means there are several ways to neutralise it before it runs:

| Method | How |
|---|---|
| **Block the network request** | Add `assets.queue-it.net` to a block list (browser extension, DevTools, hosts file). The connector script never loads; the check never fires. |
| **Inline-patch the JS** | Intercept the bundle via a proxy and comment out the queue redirect logic before it reaches the browser. |
| **Extract the secret header** | Pull the header value directly from the bundle source and call the booking APIs without ever using the site again. |

The simplest option for a one-time bypass is just blocking the domain. With `assets.queue-it.net` blocked, `/check-in` loads fully and cleanly.

---

## From Check-In to Everywhere

Here's where the SPA architecture really does the heavy lifting.

FlySafair's site is a Single Page Application. There is no traditional page-per-URL model where navigating to `/` means the browser fetches `/` from the server. Instead, once the JS bundle is running, every "navigation" is handled entirely in JavaScript: the router updates the URL bar via the History API and swaps out the rendered components  no new HTTP request to the server.

This means the Cloudflare Worker at the edge only ever sees one request: the document request that initially loaded `/check-in`. After that, it's invisible to the rest of the session.

Getting back to the home page  the booking flow entry point  is as simple as pasting two lines into the browser console:

```js
history.pushState({}, '', '/');
window.dispatchEvent(new PopStateEvent('popstate'));
```

`pushState` updates the URL bar to `/`. The synthetic `popstate` event triggers the SPA router to re-evaluate the current route and render the home page component. No GET request to `flysafair.co.za` is ever issued. The Cloudflare Worker never has another opportunity to enforce the queue.

From there, the booking flow runs exactly as it would for any paying customer. Search, seat selection, payment  all fully functional.

---

## What Actually Went Wrong

The root cause is a CDN routing inconsistency rather than a flaw in Queue-It itself.

FlySafair routes most of the site through Cloudflare with the Queue-It Worker active. But `/check-in` and `/manage` were pointed at a separate AWS CloudFront origin  probably to keep post-booking flows fast and decoupled from the sale traffic surge  and that origin was never configured behind the Worker. The queue protection has a gap, and the gap happens to serve the full site bundle.

The fix isn't complicated: every path that can serve the SPA bundle needs to pass through the same queue enforcement. Whether that's extending the Cloudflare Worker's route config or adding Queue-It's server-side KnownUser validation on the booking API endpoints, the protection needs to be consistent across the whole application rather than path-by-path at the CDN.

---

*I reached out to FlySafair on the day of the sale. I am yet to hear back from them.*
