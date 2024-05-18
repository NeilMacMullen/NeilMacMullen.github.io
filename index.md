---
layout: main
title:  "printf"
description: "thoughts on software"
---




# Latest Blog Post
{% for post in site.posts limit:1 %}
###  [{{post.title}} <br/>- *{{post.description}}*]({{ post.url | prepend: site.baseurl }})
{% endfor %}


# My projects

- [KustoLoco](https://github.com/NeilMacMullen/KustoLoco) *In-app querying using KQL*
- [Textrude](https://github.com/NeilMacMullen/Textrude) *Code-generation and text-processing utility.*
- [jumpfs](https://github.com/NeilMacMullen/jumpfs) *a tool to help developers navigate around their filesystem.*


# Blog Archive
{% for post in site.posts offset:1 limit:10 %}
  [{{post.title}} - *{{post.description}}* ]({{ post.url | prepend: site.baseurl }})
{% endfor %}




