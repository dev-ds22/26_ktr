---
title: "cicd"
layout: archive
permalink: categories/cicd
author_profile: true
sidebar_main: true
---

{% assign posts = site.categories['cicd'] %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}