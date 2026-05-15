---
title: "batch"
layout: archive
permalink: categories/batch
author_profile: true
sidebar_main: true
---

{% assign posts = site.categories['batch'] %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}