---
title: "논문 리뷰"
layout: archive
permalink: categories/papers
author_profile: true
sidebar_main: true
---

{% assign posts = site.categories.papers %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}
