---
title: Home
layout: default
nav_order: 1
permalink: /
---

# chat with ai

Claude 채팅에서 정리한 내용을 markdown으로 모아 두는 문서 사이트입니다.

## 문서 목록

{% assign groups = site.html_pages | where_exp: "p", "p.dir != '/'" | group_by: "dir" | sort: "name" %}
{% for g in groups %}

### {{ g.name | remove_first: "/" | remove_last: "/" }}

{% assign items = g.items | where_exp: "p", "p.nav_exclude != true" | sort: "title" %}
{% for p in items %}
- [{{ p.title | default: p.name }}]({{ p.url | relative_url }})
{% endfor %}

{% endfor %}

## 문서 추가 방법

1. 최상위 폴더(`claude/`, `envoy-gateway/` 등) 아래에 markdown 파일을 넣습니다. 하위 폴더도 됩니다.
2. 파일 맨 위에 `# 제목`을 적습니다. 이 제목이 사이드바와 목록에 보이는 문서 이름이 됩니다.
3. `main` 브랜치에 push 하고, Actions 탭에서 `pages build and deployment`가 성공하면 1~2분 뒤 반영됩니다.

front matter는 필요 없습니다. 순서를 직접 정하고 싶은 문서에만 맨 위에 `nav_order: 숫자`를 넣은 front matter를 추가하면 됩니다.

### 새 최상위 폴더 추가

`_config.yml`의 `defaults`에 같은 이름의 폴더 항목을 하나 추가합니다.

### 주의

front matter를 직접 넣은 문서에 Liquid 문법(중괄호 두 개, 또는 중괄호와 퍼센트 기호로 시작하는 구문)이 들어 있으면 코드 블록 안에서도 해석되어 빌드가 실패할 수 있습니다. 이런 내용은 `raw`와 `endraw` 태그로 감싸 주세요.
