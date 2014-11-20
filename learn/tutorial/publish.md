---
layout: page
title: Publishing Apps
permalink: /learn/tutorial/publish/
---

### What does it take to publish with CASA?

Not much! Just one or more folks willing to peer with you, and some JSON exposed from a web-accessible address. You'll need to tackle the former yourself, but this tutorial will walk you through the latter.

### What can be shared with CASA?

Anything that can be conveyed as a URI plus metadata. Commonly, folks share LTI tools, web apps, native apps and APIs, but there are more exotic use cases as well.

### Payload

An app is defined in CASA as an object with several key-object pairs:

1. `identity` an object that identifies the logical entity of the app;
2. `original` an object containing the properties of an app as assigned when introduced;
3. `journal` an array of changes applied as the app travelled through the network; and
4. `attributes` an object computed by applying the journal over the original attributes.

For publishing, we only need to worry about `identity` and `original`.

### Payload Identity

Because the URI of an app may change over time, CASA does not use it as the identity of an app; instead, it has the notion of a separate identity that defines which app it corresponds to.

The `identity` section of a payload is defined as:

1. `originator_id` a UUID picked by a publisher and used for all apps it publishes; and
2. `id` an string unique to the app among all apps published by the publisher.

The identity of an app might be something such as:

{% highlight json %}
{
  "id": "my-app",
  "originator_id": "cc8c99e2-fe5b-4911-a815-73c17b46d3fe"
}
{% endhighlight %}

This object will be keyed under the `identity` property of the app payload.

### Payload Attributes

Because we're publishing an app here, we only need to define the `original` attributes for an app. The `journal` will be added to by your peers as your app travels through the network, in the event than any of them elect to modify the attributes of your app.

The `original` attributes section must include the following key-value pairs:

* `timestamp` conforming to RFC 3339; and
* `uri` defining the route to the logical entity.

The `original` attributes section may also include the following key-value pairs:

* `share` set true;
* `propagate` denoting whether peers may share the node beyond; and
* `use` and `require` containing objects of key-value pairs defining attributes.

The `use` and `require` sections are where the real magic happens. The `use` section defines a set of attributes that a peer may consider when it receives an app. It may use these attributes to decide whether to keep or discard an app; for those it keeps, it will then likely use some of these attributes in how it presents the app to its users. The `require` section is similar to the `use` section, except that, if a peer does not recognize an attribute in the `require` section, it will discard the app.

What attributes are available for you to describe your app?

Some of the common ones include title, author, organization, categories and tags - but there's a catch! Because two institutions may have a different understanding of something like "ratings", instead of using human-readable names, attributes are published out by UUID keys.

Here's a basic example of the `original` section for an app:

{% highlight json %}
{
  "timestamp": "1970-01-01T00:00:01-00:00",
  "uri": "http://example.com",
  "share": true,
  "propagate": true,
  "use": {
    "1f2625c2-615f-11e3-bf13-d231feb1dc81": "Sample Application 1",
    "d59e3a1f-c034-4309-a282-60228089194e": [
      {
        "name": "Author 1",
        "email": "athor1@example.edu"
      },
      {
        "name": "Author 2",
        "email": "author2@example.edu"
      }
    ],
    "c80df319-d5da-4f59-8ca3-c89b234c5055": [
      "cat1",
      "cat2"
    ],
    "b7856963-4078-4698-8e95-8feceafe78da": "This is an example of an application published by a peer.",
    "a7e923e5-8da3-46d9-a3e8-aeda4ac8e6d5": false,
    "273c148d-de83-499e-b554-4cac9b262ab6": {
      "name": "Example University",
      "shortname": "ExU",
      "website": "http://example.edu"
    },
    "c6e33506-b170-475b-83e9-4ecd6b6dd42a": [
      "tag1",
      "tag2",
      "tag3"
    ]
  }
}
{% endhighlight %}

To help with the vocabulary:

* `1f2625c2-615f-11e3-bf13-d231feb1dc81` is a string for the app title;
* `d59e3a1f-c034-4309-a282-60228089194e` is a hash or an array of hashes for authors, each which may contain keys such as `name`, `email` and `website`;
* `273c148d-de83-499e-b554-4cac9b262ab6` is a hash for the organization publishing the app, which may contain keys such as `name`, `shortname` and `website`;
* `c80df319-d5da-4f59-8ca3-c89b234c5055` is an array of categories;
* `c6e33506-b170-475b-83e9-4ecd6b6dd42a` is an array of tags;
* `a7e923e5-8da3-46d9-a3e8-aeda4ac8e6d5` is true if the app contains explicit content, or false otherwise; and
* `b7856963-4078-4698-8e95-8feceafe78da` is a text description of an app.

These are only the most basic of attributes that exist. IMS Global's CASA Working Group is currently looking at a number of other attributes such as LTI, Caliper, Shibboleth, privacy policy, content ratings, popularity ratings, licensing, payment, device support and browser requirements, among others.

Where can you get a list of all attributes? You can't - well, not really. IMS Global will be maintaining one, in the future, but by using a UUID-based scheme, anyone may propose an attribute, and it's success is based on whether or not other institutions start regarding the attribute. This is why UUIDs are used for attributes, so that two different conceptions of a common name, or even two different implementations of the same concept, do not overlap.

### Endpoints

At minimum, CASA stipulates that you must expose an endpoint with a JSON array of all the payloads you're publishing. This endpoint is as simple as a static JSON file such as:

{% highlight json %}
[
  {
    "identity": {
      "originator_id": "1f262086-615f-11e3-bf13-d231feb1dc82",
      "id": "ex1"
    },
    "original": {
      "timestamp": "1970-01-01T00:00:01-00:00",
      "uri": "http://example.com",
      "share": true,
      "propagate": true,
      "use": {
        "1f2625c2-615f-11e3-bf13-d231feb1dc81": "Sample Application 1",
        "d59e3a1f-c034-4309-a282-60228089194e": [
          {
            "name": "Author 1",
            "email": "athor1@example.edu"
          },
          {
            "name": "Author 2",
            "email": "author2@example.edu"
          }
        ],
        "c80df319-d5da-4f59-8ca3-c89b234c5055": [
          "cat1",
          "cat2"
        ],
        "b7856963-4078-4698-8e95-8feceafe78da": "This is an example of an application published by a peer.",
        "a7e923e5-8da3-46d9-a3e8-aeda4ac8e6d5": false,
        "273c148d-de83-499e-b554-4cac9b262ab6": {
          "name": "Example University",
          "shortname": "ExU",
          "website": "http://example.edu"
        },
        "c6e33506-b170-475b-83e9-4ecd6b6dd42a": [
          "tag1",
          "tag2",
          "tag3"
        ]
      }
    }
  },
  {
    "identity": {
      "originator_id": "1f262086-615f-11e3-bf13-d231feb1dc82",
      "id": "ex2"
    },
    "original": {
      "timestamp": "1970-01-01T00:00:01-00:00",
      "uri": "http://example.com",
      "share": true,
      "propagate": true,
      "use": {
        "1f2625c2-615f-11e3-bf13-d231feb1dc81": "Sample Application 2",
        "c80df319-d5da-4f59-8ca3-c89b234c5055": [
          "cat1"
        ],
        "273c148d-de83-499e-b554-4cac9b262ab6": {
          "name": "Example University",
          "shortname": "ExU",
          "website": "http://example.edu"
        },
        "c6e33506-b170-475b-83e9-4ecd6b6dd42a": [
          "tag2",
          "tag3"
        ]
      }
    }
  }
]
{% endhighlight %}

A real world example is available at:
<br>[http://www.eduappcenter.com/api/v1/lti_apps/casa](http://www.eduappcenter.com/api/v1/lti_apps/casa)

### Well-Behaved Endpoints

In addition to the endpoint that exposes all apps, which we'll call `[path]` in this case, CASA also recommends including a route of the form `[path]/[originator_id]/[id]` for each app you're publishing; this is not required, as all the apps you're publishing are also exposed by the `[path]` route, but it is helpful to minimize packet size if a peer just wants to refresh its metadata about one app.

A well-behaved endpoint should also a valid `Content-Type` header:

```
Content-Type: application/json;charset=utf-8
```

The CASA protocol does not require either of these, so as to make it as easy to publish as just putting a properly-formatted JSON file on a web server, but both of these are highly encouraged.

### Dynamic Endpoints

For applications that already contain app metadata, when you're simply looking to publish it out with CASA as well, check out [Dynamically Publishing Apps](dynamic/).