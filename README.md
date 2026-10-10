---
title: Home
layout: default
nav_order: 1
permalink: /
---

# chat with ai

Claude 채팅에서 정리한 내용을 markdown으로 모아 두는 문서 사이트입니다.

## 문서 목록

{% for c in site.collections %}
{% unless c.label == "posts" %}
{% assign section = site.just_the_docs.collections[c.label].name | default: c.label %}
{% assign docs = c.docs | where_exp: "d", "d.nav_exclude != true" | sort: "title" %}

### {{ section }}

{% for d in docs %}
- [{{ d.title }}]({{ d.url | relative_url }})
{% endfor %}

{% endunless %}
{% endfor %}

## 문서 추가 방법

1. 저장소 루트에 `_<컬렉션 이름>/` 폴더를 만들고 markdown 파일을 넣습니다. 폴더 이름 앞의 `_`가 필요합니다.
2. 파일 맨 위에 front matter를 적습니다.
3. `main` 브랜치에 push 하고, Actions 탭에서 `pages build and deployment`가 성공하면 1~2분 뒤 반영됩니다.

```markdown
---
title: 문서 제목
---

# 본문 시작
```

사이드바 순서는 기본적으로 제목순이고, 직접 정하려면 front matter에 `nav_order: 숫자`를 추가합니다.

### 새 컬렉션(폴더) 추가

`_config.yml`의 `collections`, `just_the_docs.collections`, `defaults` 세 곳에 같은 이름을 추가해야 합니다.

### 코드 블록 주의

코드 안에 `{{ }}`나 `{% %}`가 들어 있으면 Jekyll이 Liquid 문법으로 해석해서 빌드가 실패할 수 있습니다. 이런 내용은 `raw` 태그로 감싸 주세요.
