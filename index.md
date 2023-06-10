---
layout: index.liquid
permalink: /
pagination:
  include: All
  per_page: 1
---

{% for post in paginator.pages %}
[{{ post.title }}]({{ post.permalink }})
{% endfor %}
