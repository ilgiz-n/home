---
layout: default
---

{% for post in site.posts %}
  {% assign post_title = post.title %}
  {% if post_title != nil %}
    ## [{{ post_title }}]({{ post.url }})
  {% endif %}
  
  {{ post.excerpt }}
{% endfor %}
