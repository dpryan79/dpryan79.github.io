---
layout: page
title: Devon Ryan's blog
---
{% include JB/setup %}

This is a an infrequently updated blog containing my observations from a variety of coding and data analysis projects. You can also find me on [GitHub][github], and [Twitter][twitter]. My publications are available [here][pubs].

### Posts [![](images/feed-icon-14x14.png)](rss.xml)

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

[github]: https://github.com/dpryan79
[twitter]: https://twitter.com/dpryan79
[pubs]: http://scholar.google.de/citations?user=TvpmscIAAAAJ&hl=en
