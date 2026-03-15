---
layout: category
title: DataBase
category: DataBase
permalink: /categories/DataBase/
---
{% assign count = site.categories[page.category] | size %}
{% if count == 0 %}
  <div class="empty-state">
    <h2 class="empty-state-title">(๑ > ◡ < ๑)</h2>
    <p class="empty-state-text">아직 작성된 게시물이 없습니다.<br>곧 내용이 채워질 예정입니다.</p>
    <a href="/" class="empty-state-btn">Back to Home</a>
  </div>
{% endif %}
