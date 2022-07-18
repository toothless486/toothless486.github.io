---
layout: page
title: Algorithms
permalink: /docs/algorithms/
---

# Algorithms

Welcome to the {{ site.title }} Algorithms pages! Here you can quickly jump to a 
particular page.

<div class="section-index">
    <hr class="panel-line">
    {% for post in site.docs  %}        
    <div class="entry">
    <h5><a href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></h5>
    <p>{{ post.description }}</p>
    </div>{% endfor %}
</div>
