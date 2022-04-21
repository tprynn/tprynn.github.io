---
layout: post
title:  "A Novel CSP Bypass Using `data:` URI"
date:   2019-04-11 09:00:00 -0500
categories: web xss appsec csp
---

This post originally appeared at <https://www.nccgroup.com/ae/about-us/newsroom-and-events/blogs/2019/april/a-novel-csp-bypass-using-data-uri/> but is copied here due to linkrot.


## Summary

I couldn't find an XSS payload which worked in a situation with
a very restrictive CSP. This post documents the things I tried
and the solution I ultimately found, with the help of some coworkers.

## Standard XSS Payloads Blocked By CSP

On a recent web application assessment, I ran into a DOM-based XSS[^domxss] vulnerability
which had me stumped. It was a classic DOM-based XSS: the application was returning
user data in a JSON blob, and inserting it into the page using `innerHTML`.
When `innerHTML` is used to modify the DOM, `<script>` tags don't execute[^w3],
but that's not an issue normally. Many other tags can be used, such as the
following payload using an `<img>` tag:

[^domxss]: DOM-Based XSS occurs when user input is handled unsafely in client-side
JavaScript. For example, if user input is passed to `eval()` or included into a web
page (the [DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction))
through a function that doesn't perform output encoding.
[^w3]: <https://www.w3.org/TR/2008/WD-html5-20080610/dom.html#innerhtml0>

~~~
<img src=x onerror='alert(1)'/>
~~~

In the application I was testing, though, every payload I tested was blocked
by the extremely restrictive [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP).
CSP allows a web application to whitelist sources for JavaScript (and other
resources, such as images or CSS), blocking other sources. As a result,
a strong CSP can prevent XSS even when the application itself handles user
input unsafely, by blocking inline scripts from executing at runtime.
The CSP used by the site boiled down to the following (with `myclient.com`
standing in for the domain I was testing):

~~~
Content-Security-Policy: 
default-src 'self' data: *.myclient.com;
connect-src 'self' data: *.myclient.com;
font-src    'self' data: *.myclient.com fonts.gstatic.com; 
img-src     'self' data: *.myclient.com;
script-src  'self' data: *.myclient.com;
style-src   'self' data: *.myclient.com;
report-uri /_csp; upgrade-insecure-requests
~~~

I entered the policy into Google's [CSP Evaluator](https://csp-evaluator.withgoogle.com/),
an extremely helpful tool which analyzes policies to detect weaknesses. The
obvious weakness, of course, is that this policy allows the `data:` URI.
[Data URIs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs)
allow HTML tags to be created with inline content, rather than reaching
out to and making an additional request to the server. For example, an inline image
might look like:

~~~
<img src="data:image/png;base64,iVBORw0KGgoAAAA..."/>
~~~

Since the CSP permits HTML tags to specify content via a `data:` URI, it should be
possible to embed a JavaScript payload in this way. In fact, there's plenty of
information out there about using an HTML tag with a `data:` URI
to execute JavaScript. Unfortunately, the majority of blog posts and payloads I could
find relied on the `<script>` tag - but we can't use any of them, since our payload
is included in the page using `innerHTML`. So the crux of the problem is this: find
an XSS payload which executes JavaScript from a `data:` URI, without using a
`<script>` tag. Additionally, it can't load data from untrusted domains.
(If you're the impatient type, this is where you can skip to the end of the post
to find out what ultimately worked.)

I tested a lot of payloads trying to find something that was effective. If you're
in a similar situation, I highly recommend checking out the
[PayloadsAllTheThings repo](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20injection),
but I didn't find a solution there. While experimenting, I focused in on HTML tags
which have two features: potential for JavaScript execution and the ability to set
content from a data URI. 

## Links (the `<a>` tag)

The `href` attribute of an `a` tag can be set from a data URI. However, navigation
to data URIs is disabled in
[Chrome](https://groups.google.com/a/chromium.org/forum/#!topic/blink-dev/GbVcuwg_QjM%5B1-25%5D),
[Firefox](https://blog.mozilla.org/security/2017/11/27/blocking-top-level-navigations-data-urls-firefox-59/),
and IE/Edge for a number of years.
This payload may still work in other browsers, but would require a link click. Given
those restrictions, I decided it wasn't the payload I was looking for.

~~~
<a href='data:text/html,<script>alert(1)</script>'>my link</a>
~~~

## SVG

SVGs are commonly used to bypass XSS filtering. I thought they might work here, too,
but inline scripts and event handlers I tried were unsuccessful.

~~~
<svg>
<script>alert(1)</script>
</svg>
~~~

~~~
<svg>
<g><rect onclick="alert(1)" width="300" height="300"/></g>
</svg>
~~~

~~~
<svg>
<script src='data:,utf8;alert(1)'></script>
</svg>
~~~

These SVG payloads failed for the same reasons my earlier payloads failed:
they ultimately still rely on inline JavaScript handlers (which don't pass
the CSP), or a `<script>` tag which doesn't execute with `innerHTML`. My
last attempt was embedding an SVG into the `data:` URI of an `<img>` tag,
but that doesn't execute JavaScript even if there is no CSP blocking it. 

~~~
<img src="data:image/svg+xml;utf8,&lt;svg xmlns=&quot;http://www.w3.org/2000/svg&quot; xmlns:xlink=&quot;http://www.w3.org/1999/xlink&quot; &gt;&#10;&lt;g&gt;&#10;&#9;&lt;rect onclick=&quot;alert(1)&quot; width=&quot;300&quot; height=&quot;300&quot;/&gt;&#10;&lt;/g&gt;&#10;&lt;/svg&gt;" alt="">
~~~


## `<iframe>`

With SVGs exhausted, I moved onto iframes. It was pretty straightforward to
get an iframe to render, and I controlled the style and content as well. It
probably wouldn't have been too hard to use this to turn the site into a phishing
page. However, I was having a lot of trouble getting scripts to execute, and
SOP/CSP violations when trying to navigate the frame or main window.

~~~
<iframe src="data:text/html,<html><a href=example.com>click me!</a></html>"/>
~~~

At this point, I didn't want to spend all my time testing only a single bug,
so I wrote a short email to our internal mailing list to see if any of my
coworkers had any ideas. The first break came from a coworker who suggested
trying the `async` and `defer` attributes for a script inside an iframe.
This allowed me to get scripts executing inside the frame.

~~~
<iframe src='data:text/html,<script defer="true" src="data:text/javascript,document.body.innerText=/hello/"></script>'></iframe>
~~~

However, the origin of the iframe was the [null origin](https://html.spec.whatwg.org/multipage/origin.html#concept-origin-effective-domain).
Basically, the browser treated the iframe as if it was loaded from a local file,
clearing the iframe's `document.domain`, and thus the SOP prevented communication.
This meant that JavaScript running in the frame didn't have any control over
the embedding page; I couldn't modify the page, read cookies, or anything
else I wanted to do.

After sharing those results with the thread, a second coworker came along
to provide the ultimate solution. This solution uses the iframe's `srcdoc`[^srcdoc]
attribute. The main difference in the `src` and `srcdoc` attributes, in this
case, is that `srcdoc` uses the same origin as the embedding page (when used
for an un-sandboxed iframe).

[^srcdoc]: [See documentation for the srcdoc attribute](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#attr-iframe-srcdoc)

A [Stack Overflow answer](https://stackoverflow.com/a/30507852)
provides an explanation for the different attributes. Essentially, the `srcdoc`
attribute was added along with iframe sandboxing (the `sandbox` attribute).
Older browsers, which do not support the `sandbox` attribute, would simply ignore
it. As a result, sites which relied on this attribute would fall back to insecure
behavior if they used the `src` attribute. Using `srcdoc`, on the other hand,
meant that older browsers would simply render an empty iframe if sandboxing was
not supported.

## Conclusion

So here it is, an XSS payload which:

* Triggers when written with `.innerHTML`
* Doesn't use inline JavaScript event handlers
* Doesn't load data from external domains

~~~
<iframe srcdoc='<script src="data:text/javascript,alert(document.domain)"></script>'></iframe>
~~~

I'm sure there are other payloads out there, so if you're looking for a fun
challenge, tweet me[^tw] your own solution! I bet other HTML tags like `<embed>`,
`<object>`, and `<canvas>` have some fun properties to play around with.

[^tw]: <https://twitter.com/tannerprynn>

Big thanks to Jeff Dileo and Andy Grant, who came up with the deferred
script tag and `srcdoc` solutions respectively - the client really appreciated
knowing whether this was something that could be exploited practically.
