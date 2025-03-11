---
layout: home
title: Home
nav_order: 1
description: "Flink FLIPs Blog - In-depth explanations of Flink Improvement Proposals"
permalink: /
---

# Flink FLIPs Blog

Welcome to the Flink FLIPs Blog, where we provide in-depth explanations and discussions of Flink Improvement Proposals (FLIPs).

## Latest FLIPs

{% assign sorted_flips = site.flips | sort: "flip_number" | reverse %}

<div class="flips-grid">
{% for flip in sorted_flips limit:5 %}
  <div class="flip-card">
    <h3>
      <a href="{{ flip.url | relative_url }}">{{ flip.title }}</a>
    </h3>
    <div class="flip-meta">
      <span class="label label-blue">FLIP-{{ flip.flip_number }}</span>
      {% if flip.date %}
      <span class="label label-green">{{ flip.date | date: "%Y-%m-%d" }}</span>
      {% endif %}
    </div>
  </div>
{% endfor %}
</div>

## Browse All FLIPs

You can find all FLIPs in the navigation menu on the left. FLIPs are organized by number and include detailed technical discussions, diagrams, and implementation details.
