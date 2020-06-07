---
layout: default
title: Andrea Grillo
subtitle: Personal Website & Blog
---
# About me

My name isn Andrea Grillo and I'm an electronic and telecommunication engineer. I'm from Italy, living in Genoa and currently studying. You can find me on {%- include icon-github.html -%}. 

On this page you can find some information about me and a few blog posts I wrote. Most of them are about topics that I'm most interested in.

# Latest Post

<ul style="list-style: none;">
{% for post in site.posts limit:10 %}
	<li style="color: #666">{{ post.date | date: "%-d %B %Y" }} - <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>

[Show more posts...]({% link blog.html %})