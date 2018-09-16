---
layout: page
title: Recommended resources
permalink: /resources/

resources:
  - books:
    - name: Getting things done
      description: David Allen
      url:
    - name: The Power of habit
      description: Charles Duhigg
      url:
    - name: Deep Work
      description: Cal Newport
      url:
    - name: The One Thing
      description: Gary Keller
      url:
  - websites:
    - name: Kevin Marquettes blog
      description: A great blog with a lot of valuable content      
      url: https://kevinmarquette.github.io/
    - name: The powershell subreddit
      description: Great community that offers a lot of help and let's you stay on top of what's new in Powershell.
      url: https://reddit.com/r/powershell
---    

{% for resource in page.resources %}
    <h3>{{ resource.name }}</h3>
    <ul>
        {% for section in resource.name %}
            <li>{{ section.name }}</li>
        {% endfor %}
    </ul>
{% endfor %}
