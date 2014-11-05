---
layout: page
title: Hi there!
tagline: BLogging my adventures with Java/*nix/... and ...?
---
{% include JB/setup %}

I always wanted to write something that isn't software related but at the same 
time have feeling of writing the source code. Now I just have that feeling 
as I'm writing source code of this blog . That's awesome feeling ;-) ...but 
what about unit tests etc ? You can always leave a **comment** with your personal
test result :-)

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

If you prefer reading the blog source code try [this one](https://github.com/kitarek/kitarek.github.io).

