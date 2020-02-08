---
layout: post
title:  "Filter posts by tag in Jekyll"
date:   2020-02-08 21:00:00 +0100
tags: jekyll liquid
---
To create a tag archive with [Jekyll][jekyll] we can use a `tag.html` template. This file will be responsible for listing posts with the given tag. Let's see this with a sample:

{% highlight html %}
---
layout: default
---
<div>

  <ul class="post-list">
    {% raw %}
    {% capture tag %}{{ page.title | slugify }}{% endcapture %}

    {% for post in site.posts %}
      {% if post.tags contains tag %}
      <li>
        <p class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</p>
        <h2><a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a></h2>
        {{ post.excerpt }}
      </li>
      {% endif %}
    {% endfor %}
    {% endraw %}
  </ul>

</div>
{% endhighlight %}

First, it gets the selected tag from the slugified page title, then it loops through the site posts and shows only those that contain the selected tag.

To use the `tag.html` layout, we need to create `md` files and put them into the  `tag` folder. For example, if we create a file named `jekyll.md` it would look like this:

{% highlight html %}
---
layout: tag
title: jekyll
---
{% endhighlight %}

Where title is the tag that will be filtered and layout is the `tag.html` template. [Jekyll][jekyll] will create an HTML file for each tag defined this way.

[jekyll]: https://jekyllrb.com/
