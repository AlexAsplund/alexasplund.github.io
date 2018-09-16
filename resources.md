---
layout: page
title: Recommended resources
permalink: /resources/

resources:
  - name: Books
    - name: Getting things done
      author: David Allen
    - name: The Power of habit
      author: Charles Duhigg
    - name: Deep Work
      author: Cal Newport
    - name: The One Thing
      author: Gary Keller
---    

{% for resource in page.resources %}
<h3>{{ resource.name }}</h3>
<ul>

    {% for section in resource.name %}

    <li>{{ section.name }}</li>

    {% endfor %}


</ul>
{% endfor %}
