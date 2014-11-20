---
layout: page
title: EduAppCenter Implementation
permalink: /implementations/edu_app_center/
---

This implementation (originally shared in [this gist](https://gist.github.com/bracken/6e7d23eb93be50e9dfb9)) is the sum total of code added to [EduAppCenter](http://eduappcenter.com).

The endpoint exposed by this can be found at:<br/>[http://www.eduappcenter.com/api/v1/lti_apps/casa](http://www.eduappcenter.com/api/v1/lti_apps/casa)

#### Controller Endpoint

{% highlight ruby %}
def casa

  render json: LtiApp.casas.map(&:casa), root: false

end
{% endhighlight %}

#### Model

{% highlight ruby %}
scope :casas, -> { active.published
                          .where(is_casa: true)
                          .includes(:tags)
                          .includes(:organization)
                          .includes(:lti_app_configuration) }

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