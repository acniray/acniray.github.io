---
layout: page
title: About
description: 月凉风浅
keywords: Zhang Rui, 浅月
comments: true
menu: 关于
permalink: /about/
---

很多年没写个人博客了，准备把之前的一些博文搬搬家。

04年开始编程的老码农一枚，没开源过什么项目，比较惭愧，后续会尽量做些小努力。

期待的还是诗和远方！

## Contact

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
<!-- {% if site.url contains 'mazhuang.org' %}
<li>
微信公众号：<br />
<img style="height:192px;width:192px;border:1px solid lightgrey;" src="{{ assets_base_url }}/assets/images/qrcode.jpg" alt="闷骚的程序员" />
</li>
{% endif %} -->
</ul>


## Capability

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
