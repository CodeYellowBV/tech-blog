+++
title = "APIs only a mother could love"
draft = true
slug = "ugly-apis"
date = "2017-12-15"
description = ""
aliases = ["/ugly-apis.html"]
keywords = ["Development", "API", "3rd party", "integrations"]
categories = [""]
tags = ["Development", "API", "3rd party", "integrations"]
+++

author: Peter Bex

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
being in original meaning of the acronym, this standard is anything
but simple.  Especially in dynamic languages, SOAP is a pain to
implement.  They probably got so many complaints from users of dynamic
languages that they decided to offer a
["REST"](https://en.wikipedia.org/wiki/Representational_state_transfer) interface
as well.

Unfortunately, doing REST *properly* is hard even for people
well-versed in the HTTP spec, which means when you try to do it as a
semi-automated shim around an RPC API, things will inevitably get
ugly.

For starters, almost all endpoints in this API are supposed to be
invoked via the `GET` method, *even those which modify state*!  The
HTTP specification makes a fundamental distinction
between
[methods which are safe, methods which are idempotent and methods which are neither](https://tools.ietf.org/html/rfc2616#section-9.1).

The spec is very clear that `GET` and `HEAD` are not supposed to
change things on the server (they're supposed to be "safe"):

> In particular, the convention has been established that the GET and
> HEAD methods SHOULD NOT have the significance of taking an action
> other than retrieval.

And they're certainly not supposed to do a different thing every time
you call them (they're supposed to be "idempotent"):

> Methods can also have the property of "idempotence" in that (aside
> from error or expiration issues) the side-effects of N > 0 identical
> requests is the same as for a single request. The methods GET, HEAD,
> PUT and DELETE share this property.

But, it looks like people have complained about this to the vendor,
because newer endpoints that mutate things must use the `POST` method
(or perhaps they ran into URI length limits in the GET query
params...).  Unfortunately, they kept the old API for compatibility
and still allow *only* `GET` for those.  This means it's pretty random
and you have to really check the docs because your intuition is not
going to be of any use here.


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
gears the API for downloading "new events".  It's a financial/workflow
package in which orders are entered.  When a new order comes in, we
have to process it and create a matching shop order to produce the
items.

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