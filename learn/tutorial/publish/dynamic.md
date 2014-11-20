---
layout: page
title: Dynamically Publishing Apps
permalink: /learn/tutorial/publish/dynamic/
---

**WARNING: This tutorial assumes that you have already read [Tutorial: Publishing an App](..). If you haven't, please go read that first!**

### What do we mean by dynamic?

This tutorial is meant to address the use case where you already have an application that tracks content you want to publish, and you're looking for an easy strategy to publish it out. In this tutorial, we will walk through how two Rails implementations (EduAppCenter and CASA on Rails) accomplish this.

### Mapping attributes to CASA

In EduAppCenter, Instructure stores a good deal of metadata about applications; to share with CASA, EduAppCenter needs to map its conception of metadata to CASA's attribute structure.

The `identity` structure is composed of two attributes that EduAppCenter tracks:

1. `short_name` is a unique string key that a publisher assigns to their app. This is perfect for the `id` value of the CASA payload, as its unique for the publisher.
2. EduAppCenter is an aggregator for apps published by numerous peers. Instead of putting the same `originator_id` on all of its apps, instead EduAppCenter assigns `originator_id` at the granularity of each publisher on the system. This is accessible on EduAppCenter's app model as `casa_uuid`.

This manifests as:

{% highlight ruby %}
{
  "originator_id" => self.casa_uuid,
  "id" => self.short_name
}
{% endhighlight %}

Skipping the `use` section, mapping the `original` section is easy:

{% highlight ruby %}
{
  "use" => {
    # ..
  },
  "require" => {},
  "uri" => "https://www.eduappcenter.com/apps/#{self.short_name}",
  "share" => true,
  "propagate" => true,
  "timestamp" => self.updated_at.to_datetime.rfc3339
}
{% endhighlight %}

CASA has not defined any `require` attributes yet, so that section is empty.

The only tricky part is `use` because of the fact that we need to map numerous keys to their values. The logic behind some of these...

* The title attribute `1f2625c2-615f-11e3-bf13-d231feb1dc81` is a direct mapping from the `name` on EduAppCenter.
* The tags attribute `c6e33506-b170-475b-83e9-4ecd6b6dd42a` directly maps from a `tags` relation that EduAppCenter tracks for its own use.
* The author attribute `d59e3a1f-c034-4309-a282-60228089194e` is an array of objects, each one containing several keys. Because EduAppCenter only tracks one author, it just maps `author_name`, `contact_email` and `support_url` to a single object in the array.

The others flow in a similar way:

{% highlight ruby %}
{
  "d59e3a1f-c034-4309-a282-60228089194e" => [{
    "name" => self.author_name,
    "email" => self.contact_email,
    "website" => self.support_url
  }],
  "c80df319-d5da-4f59-8ca3-c89b234c5055" => [],
  "b7856963-4078-4698-8e95-8feceafe78da" => self.short_description,
  "d25b3012-1832-4843-9ecf-3002d3434155" => self.icon_url,
  "a7e923e5-8da3-46d9-a3e8-aeda4ac8e6d5" => false,
  "c6e33506-b170-475b-83e9-4ecd6b6dd42a" => tags.map(&:short_name),
  "1f2625c2-615f-11e3-bf13-d231feb1dc81" => self.name
}
{% endhighlight %}

A twist in the attribute structure is that an app may or may not have an organization. To bypass this issue, EduAppCenter sets that key-value pair after defining the rest of the structure, and only if present. This bit, plus the entire payload definition, here:

{% highlight ruby %}
def casa

  casa_data = {
    "identity" => {
      "originator_id" => self.casa_uuid,
      "id" => self.short_name
    },
    "original" => {
      "use" => {
        "d59e3a1f-c034-4309-a282-60228089194e" => [{
          "name" => self.author_name,
          "email" => self.contact_email,
          "website" => self.support_url
        }],
        "c80df319-d5da-4f59-8ca3-c89b234c5055" => [],
        "b7856963-4078-4698-8e95-8feceafe78da" => self.short_description,
        "d25b3012-1832-4843-9ecf-3002d3434155" => self.icon_url,
        "a7e923e5-8da3-46d9-a3e8-aeda4ac8e6d5" => false,
        "c6e33506-b170-475b-83e9-4ecd6b6dd42a" => tags.map(&:short_name),
        "1f2625c2-615f-11e3-bf13-d231feb1dc81" => self.name
      },
      "require" => {},
      "uri" => "https://www.eduappcenter.com/apps/#{self.short_name}",
      "share" => true,
      "propagate" => true,
      "timestamp" => self.updated_at.to_datetime.rfc3339
    },
    "journal" => []
  }

  if self.organization.present?
    casa_data["original"]["273c148d-de83-499e-b554-4cac9b262ab6"] = [{
      "name" => self.organization.name,
      "website" => self.organization.url
    }]
  end

  casa_data

end
{% endhighlight %}

### Creating the endpoint

In EduAppCenter, they do not publish all apps via CASA; instead, this is a flag that apps may set true if they wish to publish with CASA. This means they add a new scope to the model:

{% highlight ruby %}
scope :casas, -> { active.published
                         .where(is_casa: true)
                         .includes(:tags)
                         .includes(:organization)
                         .includes(:lti_app_configuration) }
{% endhighlight %}

Finally, they define a new controller method that returns the CASA payload for all CASA-enabled apps in JSON format:

{% highlight ruby %}
def casa

  render json: LtiApp.casas.map(&:casa), root: false

end
{% endhighlight %}

### Abstracting attribute mappings

Similar to EduAppCenter, CASA on Rails maps the basic attributes right onto its representation of a payload that it will share with peers:

{% highlight ruby %}
def to_transit_payload

  if originated?

    payload = {
        'identity' => {
            'id' => payload_id,
            'originator_id' => payload_originator_id
        },
        'original' => {
            'uri' => self.uri,
            'share' => self.share,
            'propagate' => self.propagate,
            'timestamp' => self.updated_at.to_datetime.rfc3339,
            'use' => {},
            'require' => {}
        }
    }

    # .. map use and require attributes .. #

    payload

  else

    # .. ignore as this is for relaying apps it discovered from others .. #

  end

end
{% endhighlight %}

The difference is how it maps the `use` and `require` sections. Because of the extensible nature of attributes, CASA on Rails implements an architecture where attribute mappings can simply be attached to a handler, and it iterates over these:

{% highlight ruby %}
def to_transit_payload

  if originated?

    payload = {
        'identity' => {
            'id' => payload_id,
            'originator_id' => payload_originator_id
        },
        'original' => {
            'uri' => self.uri,
            'share' => self.share,
            'propagate' => self.propagate,
            'timestamp' => self.updated_at.to_datetime.rfc3339,
            'use' => {},
            'require' => {}
        }
    }

    Casa::Payload.attributes_map.each do |type, mappings|
      mappings.each do |field, klass|
        key = klass.send :uuid
        if klass.respond_to?(:make_for) # use handler's mapping function to compute payload value from app value
          value = klass.make_for(self)
        else # directly copy value from app to payload
          value = self.send(field)
        end
        payload['original'][type][key] = value unless value.nil?
      end
    end

    payload

  else

    # .. ignore as this is for relaying apps it discovered from others .. #

  end

end
{% endhighlight %}

The `each` loop for attributes stems from:

{% highlight ruby %}
module Casa
  class Payload

    cattr_reader :attributes_map

    @@attributes_map = {
        'use' => {
            Casa::Attribute::Title.name => Casa::Attribute::Title,
            Casa::Attribute::Description.name => Casa::Attribute::Description,
            Casa::Attribute::Categories.name => Casa::Attribute::Categories,
            Casa::Attribute::Author.name => Casa::Attribute::Author,
            Casa::Attribute::Organization.name => Casa::Attribute::Organization
        },
        'require' => {
            # ..
        }
    }

    # ..

  end
end
{% endhighlight %}

This delegates the attribute to a handlers specific to that attribute.

For example, consider this definition for `title`, a simple and direct mapping:

{% highlight ruby %}
module Casa
  module Attribute
    class Title

      cattr_reader :name, :uuid

      @@name = 'title'
      @@uuid = '1f2625c2-615f-11e3-bf13-d231feb1dc81'

    end
  end
end
{% endhighlight %}

In some cases, composition is more challenging than just copying from one to the other. That's why a `make_for` method is called on the handler for the app if it's defined, making it easy for the handler to do something more sophisticated. For example, consider the definition for `author`:

{% highlight ruby %}
module Casa
  module Attribute
    class Author

      cattr_reader :name, :uuid

      @@name = 'author'
      @@uuid = 'd59e3a1f-c034-4309-a282-60228089194e'

      class << self

        def make_for app
          app.app_authors.map do |c|
            data = { 'name' => c.name }
            data['email'] = c.email if c.email
            data['website'] = c.website if c.website
            data
          end
        end

      end

    end
  end
end
{% endhighlight %}

### Single app endpoint

CASA on Rails implements a controller action for the set of CASA apps that's almost identical to EduAppCenter:

{% highlight ruby %}
def all

  render json: App.where(enabled: true).map(&:to_transit_payload)

end
{% endhighlight %}

However, it also implements an endpoint for fetching only a single app:

{% highlight ruby %}
def one

  app = App.where_identity(id: params[:id], originator_id: params[:originator_id])

  unless app.nil? or !app.enabled
    render json: app.to_transit_payload
  else
    render status: 404, plain: 'Not found'
  end

end
{% endhighlight %}

The routes for these actions:

{% highlight ruby %}
get 'out/payloads', to: 'out#all'
get 'out/payloads/:originator_id/:id', to: 'out#one'
{% endhighlight %}

### And that really is it!

Go check out the [CASA on Rails source code](https://github.com/ucla/casa-on-rails) if you want to dig deeper.