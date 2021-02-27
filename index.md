---
layout: main
title:  "printf"
description: "unfiltered thoughts on software"
---

# My projects
#### [Textrude](https://github.com/NeilMacMullen/Textrude)
 *Textrude is a code-generation and text-processing utility.*

# Latest Blog Post
{% for post in site.posts limit:1 %}
###  [{{post.title}} <br/>- *{{post.description}}*]({{ post.url | prepend: site.baseurl }})
{% endfor %}



# Archive
{% for post in site.posts offset:1 limit:2 %}
  [{{post.title}} - *{{post.description}}* ]({{ post.url | prepend: site.baseurl }})

  
{% endfor %}




