---
layout: home
---

<div class="index-content concurrency">
    <div class="section">
        <ul class="artical-cate">
            <li class="on" style="text-align:left"><a href="/"><span>Concurrency</span></a></li>
            <li style="text-align:left"><a href="/scala"><span>Scala</span></a></li>
        </ul>

        <div class="cate-bar"><span id="cateBar"></span></div>

        <ul class="artical-list">
        {% for post in site.categories.concurrency %}
            <li>
                <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
                <div class="title-desc">{{ post.description }}</div>
            </li>
        {% endfor %}
        </ul>
    </div>
    <div class="aside">
    </div>
</div>
