---
title: Home
nav_order: 1
permalink: /
---
 
# chat with ai
 
Claude 채팅에서 정리한 내용을 markdown으로 모아 두는 문서 사이트입니다.
 
## 문서 목록
 
{% assign docs = site.html_pages | where_exp: "p", "p.title" | where_exp: "p", "p.nav_exclude != true" | where_exp: "p", "p.url != page.url" | sort: "path" %}
{% for p in docs %}
- [{{ p.title }}]({{ p.url | relative_url }})
{% endfor %}
