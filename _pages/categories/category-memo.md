---
title: "memo"
layout: archive
permalink: categories/memo
author_profile: true
sidebar_main: true
---

{% assign posts = site.categories['memo'] %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}