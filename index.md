---
layout: page
---
## Hello!
I'm **Navin Mohan**. Thanks for stopping by! I'm a software engineer working in the area of media streaming. 


I'm primarily interested in distributed systems.

I occasionally [blog](/blog) about things that I encounter, experiment with, or just find interesting.

### Latest Post
<ul class="post-list">
      {% assign posts = site.posts | slice:0,1 %}
      {% assign date_format = site.minima.date_format | default: "%b %-d, %Y" %}
      {% for post in posts %}
      <li>
        <span class="post-meta">{{ post.date | date: date_format }}</span>
        <h3>
          <a class="post-link" href="{{ post.url | relative_url }}">
            {{ post.title | escape }}
          </a>
        </h3>
        {%- if site.show_excerpts -%}
          {{ post.excerpt }}
        {% endif %}
      </li>
      {% endfor %}
    </ul>