+++
title = "APIs only a mother could love"
author = "Peter Bex"
slug = "ugly-apis"
date = "2018-01-24"
description = ""
aliases = ["/ugly-apis.html"]
keywords = ["Development", "API", "3rd party", "integrations"]
tags = ["Development", "API", "3rd party", "integrations"]
+++


Lately we've been integrating more 3rd party APIs than usual, and the
experience was less than great in almost every case.  Let's take a
look at how ugly some APIs will get.  In the descriptions below,
company names have been redacted to protect the (not so) innocent.

## XML is not hip, we must offer JSON

One of our 3rd party vendors has a geocoding API that is based on XML.
Of course, this data exchange format is seen as somewhat crusty and
not very hip.  So, they also offer the API in JSON.

Great, you would say. Unfortunately, the JSON responses are just
mechanically transformed versions of the original "native" XML.  That
means you can get some truly hilarious output.  Here's one small
detail in a response:

```json
{
  "AdditionalData": [
    {
      "value": "Nederland",
      "key": "CountryName"
    },
    {
      "value": "Noord-Brabant",
      "key": "StateName"
    },
    {
      "value": "Eindhoven",
      "key": "CountyName"
    }
  ]
}
```

We kind of got suckered into using the JSON API (it looked easier!),
so now our code has to normalise this so we can more easily access the
dictionary using keys:

```json
{
  "AdditionalData": {
      "CountryName": "Nederland",
      "StateName": "Noord-Brabant",
      "CountyName": "Eindhoven"
    }
}
```

In cases like this, you really wish they either created a custom JSON
format, or just stuck with the XML:

```xml
<AdditionalData key="CountryName">Nederland</AdditionalData>
<AdditionalData key="StateName">Noord-Brabant</AdditionalData>
<AdditionalData key="CountyName">Eindhoven</AdditionalData>
```

Even though this is still somewhat awkward (lots of repeated stuff),
at least you can query this easily
using [XPath](https://en.wikipedia.org/wiki/Xpath), which is a proper
XML standard (I guess the XML committee had some good ideas after
all!):

```python
country = address.findtext('AdditionalData[@key="CountryName"]')
```


## SOAP only really works in Java-like languages, so let's offer REST

In the same vein as the above, one vendor has an API that's based on
RPC via [SOAP](https://en.wikipedia.org/wiki/SOAP).  Despite "Simple"
being in the original meaning of the acronym, this standard is
anything *but* simple.  Especially in dynamic languages, SOAP is a
pain to implement.  They probably got so many complaints from users of
dynamic languages that they decided to offer a
["REST"](https://en.wikipedia.org/wiki/Representational_state_transfer)
interface as well.

Unfortunately, doing REST *properly* is hard even for people
well-versed in the HTTP spec, which means when you try to do it as a
semi-automated shim around an RPC API, things will inevitably get
ugly.

For starters, almost all endpoints in this API must be invoked via the
`GET` method, *even those which modify state*!  The HTTP specification
makes a fundamental distinction between [methods which are safe,
methods which are idempotent and methods which are
neither](https://tools.ietf.org/html/rfc2616#section-9.1).

The spec is very clear that `GET` and `HEAD` are should not change
things on the server (they're supposed to be "safe"):

> In particular, the convention has been established that the GET and
> HEAD methods SHOULD NOT have the significance of taking an action
> other than retrieval.

And they certainly should not do a different thing every time you call
them (they're supposed to be "idempotent"):

> Methods can also have the property of "idempotence" in that (aside
> from error or expiration issues) the side-effects of N > 0 identical
> requests is the same as for a single request. The methods GET, HEAD,
> PUT and DELETE share this property.

It looks like people have complained about this to the vendor, because
newer endpoints that mutate things use the `POST` method (or perhaps
they ran into URI length limits in the GET query params...).
Unfortunately, they kept the old API and still allow *only* `GET` for
those.  This means it's pretty random and you have to really check the
docs because your intuition is not going to be of any use here.


### Also, error reporting is overrated

Not implementing REST correctly is somewhat excusable (even the
"experts" can get it wrong), but what really made that same API a pain
to work with is the fact that they don't consistently implement error
handling.

For example, it supports storing a route between several waypoints.
When you save such a route, you will get a response containing XML
like `<result>123</result>`.  The `123` is an identifier you can use
when doing other operations on the route.

Usually, when something goes wrong you get a `400` error status with
an exception code and message in it (sometimes even revealing internal
server details!).  But in this particular route API, it will return
`<result>0</result>`.  This can mean anything from "required fields
are missing" to "uniqueness constraint failed".  Unfortunately, we
have a big issue with this API where it sometimes returns a zero
result for *exactly the same request* we successfully managed to send
before.  If we re-send it, sometimes it **will** work.  This makes it
impossible to debug, of course.

When we asked support about this, they simply said "zero means an
error", as if that were the most normal thing in the world....
Getting proper error messages is going to take months according to the
vendor, so we're completely stuck with debugging this, as it will fail
seemingly(!)  randomly.


## Don't worry, we'll keep track of everything for you

One API we work with uses OAuth 2.  This is its own tire fire of an
incomplete spec and inconsistent implementations.  To get a good idea
of how many issues there are, just check
the
[issue queue requests-oauthlib](https://github.com/requests/requests-oauthlib/issues),
a popular OAuth implementation for Python.  Other language
implementations seem to
have [similar issues](https://github.com/oauth-xx/oauth2/issues) with
tokens expiring seemingly without rhyme or reason.

That's bad enough as it is, and we still need to figure out a good way
of dealing with token expiry.  But one thing that really grinds my
gears is the API for downloading "new events".  It's a
financial/workflow package in which orders are entered.  When a new
order comes in, we have to process it and create a matching shop order
to produce the items.

The API is defined as follows: You pass in an identifier of your own
choosing, and you will receive from the server all the new orders
since the last request that was made with that particular identifier.
This allows for easy support of multiple servers; we have
`development`, `staging` and `production` identifiers, which means we
can test as much as we want without interfering with the production
server.

The problem with this is: what happens when an import crashes or the
connection is lost?  Then you will lose those updates, and there's no
way of retrieving them again (apart from downloading them manually)!
A better design would be to specifically request for the items you've
seen to be marked as "seen", or to tag each event with an incrementing
sequence or timestamp so that you can easily record the most recent
event you saw, and request more recent events.

### More braindeadness

One fun little detail: We had quite some trouble with getting the
OAuth implementation to work.  When we asked support about this, they
stated (multiple times) that our `application/x-www-form-urlencoded`
POST request should **not** contain url-encoded values.  Words fail me
to explain how broken this would be.  After a quick look at their
reference libraries for this API, most of them seem use URL encoding,
so it's not even true.

This is another important lesson: Bad support can really push a bad
API over the edge and make it a *terrible* one.


## So, how *does* one design a good API?

Designing good APIs is a bit of an art, and we realise that it's quite
difficult to do a good job.  One of the better books about building
good RESTful APIs
is
[Build APIs You Won't Hate](https://leanpub.com/build-apis-you-wont-hate) by
Phil Sturgeon.

And don't be afraid to stick with what you know: don't implement a new
API using a fancy new mechanism just because it's the hip thing to do,
or if you do, redesign the API from scratch.  You can always keep the
old endpoints for compatiblity, and phase it out later.

And, finally, write comprehensive documentation with good examples!
It doesn't matter how bad an API is, if the documentation is clear and
complete, a developer will know up front what to expect, and it will
hurt a lot less.

Let's wrap up with some examples of good APIs which you can study.  Of
course, this is no wall of shame, so we can finally name some names :)

### Mollie

The [Mollie payment API](https://www.mollie.com/nl/developers) is both
well-designed and extremely well-documented.  It's been years since I
used it, but it left a lasting impression of how easy and
well-documented it was.

Key points of the documentation are that there's both a reference
**and** a tutorial, the docs are split up in sections that make sense
and have flow charts illustrating the basic design.  A nice touch in
the reference are the interactive examples for making requests in
several languages.  This *includes cURL*, which is very important if
you're interested in building your own client library or just
debugging stuff at a low level.

With regards to the API itself: the most important part is that it is
versioned, which can be important for compatibility.  And, very
importantly, there's a changelog!  The API is based on proper REST
principles and supports two authentication methods: OAuth (which can
be important for fine-grained access control) and API keys (which are
hard to mess up when just getting started).

Finally, there's a *test mode* to test how payments work without
actually making money transfers. This uses the same account, so
everything else is the same, which really helps when debugging things.

### Amazon S3

The [Amazon S3 API](https://aws.amazon.com/documentation/s3/) for
storing "objects" is quite well-known and pretty popular.  So popular
in fact, that there are several third party "object store" providers
which implement this API on top of their own backend. I've recently
dealt with such a third-party implementation to implement an [S3
backend](https://wiki.call-cc.org/eggref/4/ugarit-backend-s3) for
[Ugarit](https://www.kitten-technologies.co.uk/project/ugarit/doc/trunk/README.wiki),
which is a fascinating backup system.  In doing so, I had to update
the CHICKEN [S3 library](http://wiki.call-cc.org/eggref/4/amazon-s3) a
bit to make it work with the third-party provider I was using at the
time, so I got to know the API a bit.

The first thing I noticed about the S3 API is that it is truly *vast*;
it offers tons of bells and whistles.  Most people won't need to use
those.  Luckily you can ignore a lot of that complexity when you don't
need it.  It won't get in your way when you're not using it; that's
good design!

I only don't like much how authentication is handled.  You have to
*sign* your request, which requires you to craft a sort of
["canonicalized"
request](https://docs.aws.amazon.com/general/latest/gr/sigv4-create-canonical-request.html).
There are a lot of ways this can go wrong, especially because you to
know the headers that will be sent.  This may require some hackery to
make sure this lines up with the headers that your HTTP client will
(automatically) send.

The documentation is extensive, but it doesn't seem very friendly and
it feels very *scattered*, probably due to the sheer size of the API.
One aspect of good API design is knowing the goals you're trying to
accomplish and setting limits on what you're willing to support in the
API.  Not every API needs to support every conceivable thing;
consistency, correctness and ease of use are more important.

The documentation is also a bit vague about the pitfalls of this API;
later I learned that (in the Amazon implementation at least) the store
is *eventually consistent*.  This may mean in practice that after you
store an object, you might not see it in a listing and retrieval might
fail.  You might also get an earlier version of the object.  This has
a massive impact on the design of any system that uses the API, so the
docs could be a bit clearer about this.  The [introduction to
S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/Introduction.html)
explains this, but the API reference does not mention it even once.
