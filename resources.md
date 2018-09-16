---
layout: page
title: Recommended resources
permalink: /resources/


books:
- name: Getting things done
  description: David Allen
  url: https://en.wikipedia.org/wiki/Getting_Things_Done
- name: The Power of habit
  description: Charles Duhigg
  url: https://en.wikipedia.org/wiki/The_Power_of_Habit
- name: Deep Work
  description: Cal Newport
  url: http://calnewport.com
- name: The One Thing
  description: Gary Keller
  url: https://en.wikipedia.org/wiki/The_ONE_Thing_(book)
websites:
- name: Kevin Marquettes blog
  description: A great blog with a lot of valuable content      
  url: https://kevinmarquette.github.io/
- name: The powershell subreddit
  description: Great community that offers a lot of help and let's you stay on top of what's new in Powershell.
  url: https://reddit.com/r/powershell
---    

## Blogs and websites
{% for section in page.websites %}
#### [{{ section.name }}]({{ section.url }})

{{ section.description }} 
* [{{ section.url}}]({{ section.url}})

<br />

{% endfor %}

---

## Books
{% for section in page.books %}


#### {{ section.name }}

* Author: {{ section.description }}
* [{{ section.url}}]({{ section.url}})



{% endfor %}
