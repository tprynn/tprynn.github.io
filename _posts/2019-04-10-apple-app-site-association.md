---
layout: post
title:  "Apple's App-Site Association - The New `robots.txt`"
date:   2019-04-10 09:00:00 -0500
categories: web recon
---

This post originally appeared at <https://www.nccgroup.com/ae/about-us/newsroom-and-events/blogs/2019/april/apples-app-site-association-the-new-robots.txt/> but is copied here due to linkrot.


## Summary

Apple created App-Site Associations, which requires websites to host a file
`.well-known/apple-app-site-association`. This file contains a bunch of
information on web app routes like `robots.txt` from the good-old days.

## The New `robots.txt`

Long gone are the days where [`robots.txt`](https://en.wikipedia.org/wiki/Robots_exclusion_standard)
was a useful source of information disclosure about web applications. Outside of
some beginner-level CTFs, it's unlikely you'll find any useful routes at
`/robots.txt` that won't be easily found with Burp's crawler or a tool like
[DirBuster](https://tools.kali.org/web-applications/dirbuster).

Enter Apple's **App-Site Association**. This iOS-specific standard allows companies to
link their iOS applications to their domain in order to support functionality like:

* When users click mobile links in iOS Safari, the company's mobile app
can intercept the link click rather than navigating to the mobile site[^links].

* Sharing user credentials between multiple iOS applications developed by
the same company.

* Using Handoff[^handoff] to seamlessly transfer activity between multiple iOS devices
owned by the same user.

[^links]: [Apple's documentation for Universal Links](https://developer.apple.com/library/archive/documentation/General/Conceptual/AppSearch/UniversalLinks.html)

[^handoff]: [Apple's documentation for Handoff](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/Handoff/HandoffFundamentals/HandoffFundamentals.html)

But how can an App-Site Association help us test web applications? The answer
is that iOS requires a two-way trust between an iOS application and a domain
in order to set up this type of association. Both the iOS application and domain
have to opt in to enable the trusted relationship.

For the iOS application, developers set the desired associated domains in the
application's configuration at compile time. At runtime, the user's device
reaches out to the configured domain, which must respond with a JSON blob
containing permitted app identifiers and URLs to open in the corresponding
app. This JSON blob must be hosted at the hardcoded route
`/.well-known/apple-app-site-association`.

As a result, pretty much any site with a corresponding iOS application
hosts this publicly-accessible file listing, which lists site URLs that
should be opened in the mobile app. For example, Apple's own domain hosts
an app association at <https://www.apple.com/.well-known/apple-app-site-association>:

~~~
{
  "activitycontinuation": {
    "apps": [
      "W74U47NE8E.com.apple.store.Jolly"
    ]
  },
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "W74U47NE8E.com.apple.store.Jolly",
        "paths": [
          "NOT /shop/buy-iphone/*",
          "NOT /us/shop/buy-iphone/*",
          "/xc/*",
          "/shop/buy-*",
          "/shop/product/*",
          "/shop/bag/shared_bag/*",
          "/shop/order/list",
...
~~~

This file specifies the app ("Jolly", the Apple Store application), along with
dozens of URLs which should or shouldn't be opened - in a very similar way to
the old `robots.txt`. Especially on black box tests or for bug bounty work,
this file can help testers to find hidden routes or build a word list for
further testing. On many sites, the file also contains app identifiers and
URL lists for internal and enterprise apps, further expanding the amount of
information which is disclosed.

## Takeaways

*For penetration testers*: Doing black box web testing or bug bounty work?
Forget `robots.txt`, and start checking `.well-known/apple-app-site-association`.

*For web and mobile developers*: If you're using Apple's App-Site Association,
be aware that all the information you publish there is public. Check this file
on your sites, and make sure you aren't leaking anything you don't want the world
to know, such as internal URLs and development or enterprise applications.
