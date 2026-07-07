# Steam Overlay Browser — Sandbox Escape & Session Data Exfiltration

One of my first security writeups, so bear with me if it's a little rough.

---

## How I got here

I was messing around with the Steam overlay browser one day — the in-game one you get with Shift+Tab. I already knew it blocked outbound network requests, `fetch` and `XMLHttpRequest` both just silently fail when you try to hit an external host. Valve obviously put that in there intentionally. But I was curious whether it was actually enforced properly or just bolted on.

Turns out, it's bolted on.

---

## The Sandbox

So the overlay browser is essentially a stripped-down Chromium running inside the Steam client. When you open it in-game and browse to a Steam page, it blocks outbound requests to non-Steam hosts. Try running this in the console on a Steam page in the overlay:

```javascript
fetch("https://example.com").then(r => console.log(r))
```

Nothing. Same with `new XMLHttpRequest()`. The requests just don't go anywhere.

At first I thought this might be something enforced at the Chromium level — a CSP header or a sandbox flag. But looking closer, it seemed more like a network-layer intercept. The restrictions felt like they were attached to the browsing context of the original page rather than baked into the browser itself.

That gave me an idea.

---

## `document.write()` and the Context Reset

If the restrictions are tied to the page's browsing context, what happens when you blow up the context entirely?

`document.write()` does exactly that. When you call it, it tears down the current document and replaces it with whatever you write. New DOM, new scripts, effectively a new page. I figured if the sandbox restrictions were tied to the original context, a `document.write()` call might leave them behind.

You can invoke `document.write()` through a `javascript:` URI in the URL bar:

```javascript
javascript:document.write('<html><body><script>
  fetch("https://example.com")
    .then(r => r.text())
    .then(t => console.log(t));
<\/script></body></html>');
```

And it worked. The request went straight through.

`XMLHttpRequest` works too, and so does dynamic script loading — you can inject a `<script src="...">` pointing at an external host and it loads and executes fine. The rewritten page has none of the restrictions the original had.

---

## The URL Filter

When I first tried this with a raw URL in the payload, it didn't work — the request was blocked before it even left the browser. Valve has some kind of pattern matching on URLs in `javascript:` URIs.

Getting around it was trivial though. Reversing the URL string and reversing it back at runtime:

```javascript
var u = "moc.elpmaxe//:sptth".split("").reverse().join("");
fetch(u);
```

That's enough to slip past it. The core issue isn't the filter anyway — it's that `document.write()` escapes the sandbox entirely.

---

## What's Accessible From the Escaped Context

Once you have unrestricted code execution in the overlay browser, a few things become interesting.

**Cookies.** The overlay browser is browsing Steam's own pages, so it carries the user's active session cookies. Some of them — `sessionid`, `clientsessionid`, `browserid` — are not marked `HttpOnly`, so they're readable via `document.cookie`. Worth noting: `sessionid` on Steam is a CSRF token, not the actual session authenticator. Exfiltrating it on its own doesn't get you account access. But it's still readable, and it's the kind of thing that shouldn't be.

**WebAPI token.** This is more interesting. The `/account` page embeds a `webapi_token` JWT in a `data-store_user_config` attribute in the HTML. Since the escaped context can make credentialed XHR requests to Steam's own pages, you can pull this token directly:

```javascript
var x = new XMLHttpRequest();
x.open("GET", "/account", false);
x.withCredentials = true;
x.send();

var idx = x.responseText.indexOf("webapi_token");
if (idx > 0) {
  var token = x.responseText.substring(idx, idx + 600);
  // exfiltrate token
}
```

This JWT works externally, from any machine, and stays valid for around 24 hours. With it you can hit the Steam Web API as the authenticated user — private friends list, chat history, trade history, points balance, notifications. The actual interesting data.

**The SteamClient bridge.** The overlay browser exposes a `SteamClient` JavaScript object that talks directly to the Steam client process. From the escaped context this is reachable. A few things you can call on it:

- `Browser.OpenDevTools()` — opens Chrome DevTools inside the overlay, which exposes all cookies including the `HttpOnly` ones
- `Browser.Paste()` — reads the system clipboard
- `ClearAllBrowsingData()` — wipes the browser session

---

## Chaining With XSS

On its own this needs someone to paste a `javascript:` URI into the URL bar, which limits it. But the overlay browser is used to browse Steam community pages — profiles, workshop items, guides, forum posts — all of which accept user-generated content. If there's a stored XSS anywhere on those surfaces, the whole thing becomes zero-interaction for anyone who views that page in the overlay.

I didn't go looking for a Steam XSS to chain it with. Not my goal here. But the attack surface is large.


## Takeaways

The sandbox bypass itself is the main finding — `document.write()` fully escapes the network restrictions because those restrictions are attached to the browsing context rather than the browser configuration. That's the architectural issue. Everything else (cookie exposure, JWT extraction, bridge access) flows from that.

The URL filter is trivially bypassed and probably shouldn't be relied on for anything. If the sandbox itself is sound, the filter doesn't matter. If the sandbox isn't sound (which it isn't), the filter doesn't help.

Anyway. Interesting rabbit hole.
