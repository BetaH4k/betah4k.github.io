---
layout: archive
permalink: /codigo-seguro/
title: "Código seguro"
author_profile: true
---

{% assign tag_filter = "codigo seguro" %}
{% assign filtered_posts = "" | split: "" %}

{% for post in site.posts %}
  {% assign tags_normalized = post.tags | join: "," | downcase | split: "," %}
  {% if tags_normalized contains tag_filter %}
    {% assign filtered_posts = filtered_posts | push: post %}
  {% endif %}
{% endfor %}

<div class="tagged-posts">
    <h2 id="{{ tag_filter | slugify }}" class="archive__subtitle">{{ tag_filter }}</h2>
    {% if filtered_posts.size > 0 %}
        {% for post in filtered_posts %}
            {% include archive-single.html %}
        {% endfor %}
    {% else %}
        <p>No posts found with the tag '{{ tag_filter }}'.</p>
    {% endif %}
</div>
