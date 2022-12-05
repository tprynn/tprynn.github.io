---
layout: post
title:  "Why *NOT* To Pin TLS Certificates"
date:   2022-12-06 09:00:00 -0500
---

> "Certificate pinning is the easiest way to take your least pressing security problem and turn it into your most pressing operational problem."

\- Someone I misquoted on Slack

The discussion of whether to implement TLS certificate pinning (primarily for mobile applications) is one that seems to come up with annoying regularity. Among my circle of modern appsec practitioners, the consensus is that pinning is a bad idea. However, there's no real canonical post I can link to about *why* it's bad. So here's that post.


## Pinning is a Footgun

The first and most important reason not to pin is because it's a massive footgun. If your certificate breaks, your application is now broken for all of your users. To fix it, you need to publish a new app version (could take days) and wait for all your users to update (could take weeks or months, assuming they don't just uninstall your app).

Your certificate can break in dozens of un-fun ways (it expires, you need to revoke it, you lose your private keys, ...), and certificate pinning adds new ones (you change domains or CAs, you mess up your configuration), while making recovery hard or impossible. This complexity is exactly [what killed HPKP](https://www.zdnet.com/article/google-chrome-is-backing-away-from-public-key-pinning-and-heres-why/), the pinning mechanism for browsers.

To avoid the footgun, the best advice you can probably find on the Internet says to pin to a CA rather than a leaf certificate. You're still left with most of the footgun and you've probably lost most of the benefit, but I'll leave you to weigh whether it's worth it. You probably won't be able to find advice on what the best and most modern approaches are such as whether to pin keys vs certificates, how to use unique hostnames per-version or platform of your app, and making sure you have a fallback chain for break-glass scenarios.

If someone does write a guide describing how to do certificate pinning *effectively*, maybe the calculus changes. But probably not, because operationalizing it is just way too complicated. I'm not even saying it's not possible, but the advice *doesn't even exist for you to follow*! No one has written it down. The only way you'll get it right is if you hire someone who has already done it and figured out all the ways it can go wrong.


## Absence of a Logical Threat Model

Pinning doesn't make sense for 99% of companies and apps. (If you're in the 1%, feel free to ignore this post and have a thorough discussion with your 100+ person security team and 1000+ engineers.) The arguments to do so are based on purported threats which lack a rational foundation. Here are the facts which I base my threat model on:

1. You are overwhelmingly unlikely to be targeted by a sophisticated attack involving a compromised CA.
2. The ecosystem-level protections of [Certificate Transparency](https://certificate.transparency.dev/) and OS/browser CA Root teams have made such attacks vastly more costly, less effective, and easier to respond to. You can further use [CAA records](https://en.wikipedia.org/wiki/DNS_Certification_Authority_Authorization) to limit which CAs issue certificates for you domains.
3. In the case where you are targeted by such an attack, the value of certificate pinning is purely marginal because you most likely have a browser site where all the same functionality is available but certificate pinning is not.
4. You most likely applied pinning only to one of your dozens of domains that handle sensitive info and you're probably not even validating TLS certificates at all for the [connection between your web app and the database](https://innerjoin.bit.io/the-majority-of-postgresql-servers-on-the-internet-are-insecure-f1e5ea4b3da3).

Other arguments for pinning include device-level compromise ("my user's device is rooted with malware that can install a new CA") and obfuscation ("pinning makes it harder to reverse engineer"). I'd need two more posts to go into depth on these, but in a few words:

1. If the user's device is compromised by root-level malware (or by the user themselves), there is nothing you can do to protect your own app from that attack.
2. Arguing for obfuscation is fundamentally the same as arguing for [security through obscurity](https://en.wikipedia.org/wiki/Security_through_obscurity). You probably won't even waste five minutes of someone's time  given off-the-shelf anti-pinning tools ([1](https://npm.io/package/apk-mitm), [2](https://httptoolkit.com/blog/frida-certificate-pinning/), [3](https://github.com/sensepost/objection)).

To briefly revisit the 99% vs 1% argument: I'm not saying there is *no value at all* to pinning. But the arguments to pin don't even start to make sense unless the marginal benefit of pinning represents a significant difference in the outcomes for your users and your company - i.e. you're a top-10 financial institution or you're Google, Apple, a government, likely to be targeted by a government, etc.


## Security is Cost vs. Benefit

There is no company in the world that has zero security risk. Even the companies that spend by the far the biggest amount of time and effort on security (in both absolute and relative terms) still regularly discover and fix vulnerabilities. I have worked and consulted with at least a hundred companies across many industries and I can't recall a single one where I thought "certificate pinning is in the top 10 most valuable things you could work on".

If your job title is anything like "security engineer", you will constantly be full up on possible projects and opportunities to improve security. The only way to make sense of those is to apply some kind of prioritization based on the value you think they'll bring you against the amount of effort required to implement them.

Somewhere far down the priority list, you start getting into the territory of projects that actually leave you worse off then you started. These boondoggles end up creating more work for you dealing with all kinds of unrelated problems that you didn't even have when you started, taking up time you could have used to make meaningful changes. It's up to you to identify and avoid this trap! And I do mean *you*, actively, because external standards simply don't provide this guidance. I'm not trying to put them on blast, but OWASP places ["Code Tampering"](https://owasp.org/www-project-mobile-top-10/2016-risks/m8-code-tampering) and [Reverse Engineering](https://owasp.org/www-project-mobile-top-10/2016-risks/m9-reverse-engineering) within the same threat framework to [not using TLS at all](https://owasp.org/www-project-mobile-top-10/2016-risks/m3-insecure-communication).

## Rant Over

When you find yourself thinking about what security projects you should work on, you have to consider the universe. The actions you take and controls you implement don't exist in a vacuum. If the threats don't make sense, if the cost is too high, the priority won't meet the bar.

**Go work on something that matters** like implementing a strong CSP or adding a static analysis rule to check for missing auth or improving your audit logs or reducing the attack surface of your support web app or making all your HTTP requests go through [smokescreen](https://github.com/stripe/smokescreen) or adding new threat detections or reducing the privileges of your IAM accounts or getting all your employees to use strong two factor auth or detecting leaked API keys or ...