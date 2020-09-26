---
layout: default
title: Welcome Page
---

## Welcome, travellers, to the cyber corner of a Reverse Engineer. Here be good ale, and dirty analyses of binaries that I find interesting!

Here are the entries so far: 

<div class="posts">
  {% for post in site.posts %}
    <article class="post">

      <h2><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h2>

      {% comment %}
      <div class="entry">
        {{ post.excerpt }}
      </div>
      
      
      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">Read More</a>
      {% endcomment %}
    </article>
  {% endfor %}
</div>



