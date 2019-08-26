# PIGIN: Private Interest Groups, Including Noise

## Introduction

Some online advertising today is based on showing an ad to a person who has previously interacted with the advertiser.  Today this works by the advertiser recognizing a specific person as they browse across web sites, a core privacy concern with today's web.

We propose an API in which the browser, not the advertiser, holds onto the information about what the advertiser thinks a person is interested in.  Shifting this data to the browser instead of servers lets us offer many advantages: clear privacy properties, time limits, transparency into how interest groups are built and used, and granular or global controls over this type of ad targeting.

This type of ad targeting can be of value to people browsing the web, who often prefer ads for things they are interested in, and to advertisers, who wish to show their ads to people more likely to be interested in them.  That added value is a significant part of why publishers [earn much more money today when cookies are available](https://www.blog.google/products/ads/next-steps-transparency-choice-control/) for ad targeting.  Therefore an API that supports this practice and offers privacy guarantees is an important part of creating a web in which publishers can flourish without cross-site tracking.

This proposal will not support _all_ related use cases — for example, we require an advertiser's interest group to be larger than some threshold size, to avoid "microtargeting" in which the API itself could become a tracking mechanism.  Nevertheless it shows how an important web advertising use case can be made compatible with strong privacy protections for people's browsing behavior.


## Motivating Use Case

An "interest group" is a collection of people whom an advertiser believes will be interested in seeing some type of ad.  As people browse and interact with a web site, the site operator can add people to any number of interest groups.  Advertisers then use those interest groups for ad targeting, especially to entice past visitors to return to the site.  The advertising industry today uses a variety of terms to refer to variations on the idea of what we're calling an interest group, including "user list", "remarketing list", "custom audience", and "behavioral market segment".

As a concrete example, consider retailer WeReallyLikeShoes.com who sells shoes on the web. When they first show up at the web site, the visitor will be associated with a "WeReallyLikeShoes-shopper" interest group. As they view different sections of the site, they will be associated with the "WeReallyLikeShoes-athletic-shoes" or "WeReallyLikeShoes-dress-shoes" interest groups. When they view particular shoes, they will be associated with the "WeReallyLikeShoes-shoe-00123-viewer" interest group. Finally, purchasing the shoe or leaving the website will associate them with either the "WeReallyLikeShoes-buyer" group or "WeReallyLikeShoes-cart-abandoner" group.

Later, WeReallyLikeShoes.com wants to run an ad campaign for potential shoe buyers. They could use their interest groups for more precise targeting, such as offering a discount to members of their "cart-abandoner" group or advertising to their "athletic-shoes" group at the beginning of summer.

The goal of the API is to support this use case with the following outcomes:

*   People who like ads that remind them of sites they're interested in can keep seeing those sorts of ads.
*   People who don't can avoid seeing those sorts of ads.
*   People who wonder "how the ad knew" what they were interested in can get a clear, accurate answer.
*   People who wish can sever their association with the interest group and stop that advertiser from targeting them.
*   Advertisers cannot learn the browsing habits of any specific people, even ones who have joined multiple interest groups.

All details of the UI would, of course, be up to the browser.  The API provides the infrastructure that enables this ad campaign targeting capability while enforcing privacy rules that make transparency and control possible.


## Design Elements

In this proposed API there is no server that keeps track of which people are in what interest groups.  Instead, when an advertiser sees an interesting action, we let them ask the browser "Please join my WeReallyLikeShoes-athletic-shoes interest group for the next 30 days".  Then on some future web pages with ads, the browser might choose to tell the ad server "By the way, I'm in the WeReallyLikeShoes-athletic-shoes interest group."

Interest group memberships remain private unless the browser chooses to disclose one.  The browser has a way to be sure that a group membership is "anonymous enough" before choosing to disclose it.  There is no way to ask the browser "Are you in my buyer group?" or "What are all the interest groups you're in?"


### Browsers Joining Interest Groups

There is a straightforward JS API for an advertiser asking a browser to join a particular interest group for some amount of time.

```
var myGroup = {'owner' : 'www.wereallylikeshoes.com',
               'name' : 'athletic-shoes',
               'readers' : ['first-ad-network.com',
                            'second-ad-network.com']
              };
window.navigator.joinPrivateInterestGroup(myGroup, 30 * kSecsPerDay);
```

The API must be called from a window (top-level or iframe) whose origin matches the ``owner``.  This could be on WeReallyLikeShoes.com, or could be a cross-domain iframe — maybe RunningShoeReviews.com writes articles about shoes sold by WeReallyLikeShoes.com, and the review site has an agreement which lets the retailer add people to an interest group with ``'name' : 'reads-reviews'``.  It should also be possible for a site owner to include a cross-domain iframe _without_ giving it this capability.

The browser will only consider revealing group membership on requests to ``reader`` domains.  This is meant to protect the owner's business interests, and is an extra limit on top of whatever the browser imposes to protect privacy.  (Perhaps we should add support for pass-through domains: a way for ``first-ad-network.com`` to indicate that it buys ad-displaying opportunities from ``some-other-ad-platform.com``, including a public encryption key so that the browser has a way to pass the group membership information through untrusted channels.)

If an interest group owner needs to know multiple pieces of information to decide whether they'd like a person to join their interest group, then the owner is responsible for tracking all the membership conditions on their own (on their server or in browser storage).


### Browsers Disclosing Interest Groups — the tricky part

When making an HTTPS request to a domain that is one of the ``readers`` of any of the interest groups a browser has joined, the browser may choose to include information about one or more of those interest groups in an HTTP request header.

```
GET https://first-ad-network.com/serve_ad.html?width=300&height=250
Referer: https://somelocalnewspaper.com/big-story.html
Sec-CH-PIGIN: www.wereallylikeshoes.com:athletic-shoes
Sec-CH-PIGIN: www.rundontwalk.com:repeat-customer
```

Those interest group memberships can then be used by the ad network in picking which ad to show.

The browser's key responsibility is figuring out what set of group memberships it can disclose while preventing tracking and respecting privacy.  Its secondary goal should be to send the "most valuable" interest group information that meets its privacy threshold.  

At a minimum, the set of interest groups that the browser chooses to disclose should be the same for many different people who might visit a website — that is, a _k_-anonymity threshold.  This is necessary so that nobody can use a person's disclosed interest groups as a way to recognize them and track their browsing behavior across sites.  Research in differential privacy offers a variety of guarantees to consider that are stronger than simple _k_-anonymity.

In any case, ensuring that a collection of interest groups is sufficiently private is likely to involve communication with some privacy infrastructure servers.  The browser's communication with them should again protect privacy and not enable tracking — preferably, even by the operators of the privacy infrastructure.  (For example, a browser sending a complete list of interest groups to a "private subset picking service" could leak someone's identity and full browsing history to the service.)

**Picking or approximating the "most valuable" sufficiently-private set of interest groups is a key challenge.  This topic should be the subject of discussions between browsers and the advertising industry** — an industry with a wealth of experience at picking the most valuable ad.

Understanding which interest groups are the most valuable might involve periodic out-of-band interactions between the browser and additional servers run by various ad tech companies.  Any such communication with them must again be designed to protect privacy.

Note that privacy should be preserved even if all the reader domains that co-appear on a site collaborate with each other.

### Naive list-picking example

For example, a naive implementation could consist of a daily round of decision-making:

*   Contacting a server run by each reader domain to learn the value of each interest group, using Private Information Retrieval techniques to avoid the need to disclose any list memberships to the reader domain.
*   Sorting interest groups by highest declared value.
*   Querying a server, using Threshold Cryptography, to learn whether at least 1000 other people have the same most-valuable interest group.  If not, discard that group and repeat.
*   If so, accept the top interest group, and consider whether it is OK to send the next most valuable group as well.  Query the Threshold Crypto server again, this time asking about the top two groups together.
*   Repeat until the browser has chosen a maximum of 5 interest groups or checked all groups.
*   Ad requests during the following day get the subset of the chosen interest groups for which the ad network is a permitted reader domain.

This can be improved on in many ways.  Simple _k_-anonymity is an exploitable notion of privacy, and the value of an interest group to a reader domain may vary depending on what web site the browser is visiting.  An implementation in which the browser may send a different set of interest groups while visiting different sites could improve both privacy and advertising value.  The threat of reader domain collaboration is addressed by using a global ordering of interest group value, rather than permitting a different value for different readers; this is bad since readers might lie to cause harm to other readers.





### User Interface controls

This API means browsers can offer people a UI that provides insight into what interest groups they are on and how they got there, as well as control over both past and future group memberships.  Some ideas for controls that a browser might choose to offer include:



*   "What interest groups am I in?" — Show the owners, names, and expiration dates of interest groups.  Perhaps browsers should also require interest group owners to give a URL of a sample image from an ad campaign targeting any group.
*   "How did I get on this list?" — The display of any interest group should include the advertiser's domain name, and also the day and domain of the top-level page the browser was visiting when it joined that group.
*   "Remove me from this list" — Tell the browser to leave any or all interest groups.
*   "Block this advertiser" or "Block this site" — The browser could leave all interest groups owned by WeReallyLikeShoes.com or all interest groups joined while on RunningShoeReviews.com, and disallow such list additions in the future.
*   "Disable Private Interest Groups" — This control is analogous to disabling 3rd-party cookies today, but without the disadvantage of breaking other unrelated use cases.


## Privacy and Security Considerations

This API allows one site very limited information about a visitor's off-site interests.


### Interest group membership indicating a sensitive category

If a person visits a web site and that site learns/decides that they are in some sensitive category, the site could build an interest group of sensitive category members, and could perhaps make that information available even when the person is visiting a different site.

This is partly mitigated by the _k_-anonymity (or stronger) requirements on what interest groups the browser reveals, but some sensitive categories may be large.  It is also partly mitigated by only revealing the "high-value" group memberships, but a well-resourced malicious site could genuinely run a high-priced ad campaign to encourage the browser to disclose this sensitive-category signal.

Differential privacy properties for the revealed interest groups could be chosen to offer some measure of plausible deniability — noise of the "false positive" variety, in addition to the false negatives from only revealing some interest groups.  However, note that this would cause some advertisers to incorrectly target their ads.


### Collection of a repeat visitor's profile over time

If a person visits a particular first-party site over a long period of time, then that site may have a stable ID for the person, e.g. a first-party cookie or a logged-in account.  Interest groups are only revealed to reader domains, but the first party could also be in the ads business, or could collaborate with the readers who provide its ads to record the history of the person's interest groups revealed by the browser.  Even if each day's ad requests offer appropriate privacy guarantees, a first-party site can build knowledge over time.

If the publisher, advertiser, and ad networks are all willing to collaborate, this seems technically difficult to prevent.  Even if browsers invented a way to render certain ads on a site while making it impossible for a collaborating ad to tell the site which ad was shown, many server-to-server communication schemes could ultimately share the same information.


### Tracking a person's browsing across sites

This is partly mitigated by the _k_-anonymity (or stronger) requirements on what interest groups the browser reveals.  However, the "collection of a repeat visitor's profile over time" strategy will cut down on that anonymity.  Two different sites that the same person visits frequently could use this to make guesses about their matching visitors.

This would be partly mitigated by the browser sending different choices of interest groups to different sites.  The timeline for this attack would be extended by browsers rotating interest groups more slowly.  But this cannot be solved completely: If a person visits two different sites often enough for long enough, this is just one of many signals which the sites could use to correlate behavior patterns and try to pick out matching visitors.
