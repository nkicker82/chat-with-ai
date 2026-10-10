{% assign docs = site.pages | where_exp: "p", "p.dir == '/docs/'" | sort: "path" %}
<ul>
{% for p in docs %}
  {% unless p.name == "index.md" %}
  <li><a href="{{ p.url | relative_url }}">{{ p.title | default: p.name }}</a></li>
  {% endunless %}
</ul>
