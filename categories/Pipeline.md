---
layout: category
title: PipeLine
category: PipeLine
permalink: /categories/pipeline/
---
{% assign count = site.posts | where_exp: 'post', 'post.categories contains "PipeLine"' | size %}
{% if count == 0 %}
  <div class="empty-state">
    <h2 class="empty-state-title">(๑ > ◡ < ๑)</h2>
    <p class="empty-state-text"><br>곧 내용이 채워질 예정입니다!</p>
    <a href="/" class="empty-state-btn">Back to Home</a>
  </div>
{% endif %}
