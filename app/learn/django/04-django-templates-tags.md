<!-- ramani: app/learn/django/04-django-templates-tags.md -->
⬅️ [Back to Backend Notes](README.md)

---

# Django: Templates & Template Tags

## Contents

- [How Templates Work](#how-templates-work)
- [Context](#context)
- [Context Processors](#context-processors)
- [Template Inheritance](#template-inheritance)
- [Built-in Tags](#built-in-tags)
  - [url](#url)
  - [static](#static)
  - [include](#include)
  - [if / elif / else](#if--elif--else)
  - [for](#for)
  - [with](#with)
  - [Other Useful Tags](#other-useful-tags)
- [Built-in Filters](#built-in-filters)
- [Custom Template Tags](#custom-template-tags)
  - [Setup](#setup)
  - [simple_tag](#simple_tag)
  - [inclusion_tag](#inclusion_tag)
  - [Passing Context to Tags](#passing-context-to-tags)
  - [Custom Filter](#custom-filter)
- [Template Partials](#template-partials)
- [References](#references)

---

## How Templates Work

A template is an HTML file with placeholders and logic. Django's template engine fills in those placeholders with data passed from a view, then returns the result as a plain HTML string.

```
View builds a context dict  →  Template engine renders  →  HTML string  →  HttpResponse  →  Browser
{"article": <Article>}          article.html + context
```

Django's template engine searches two places, in order:

1. Directories listed in `TEMPLATES[0]["DIRS"]` — project-level templates
2. Each app's `templates/` folder (when `APP_DIRS: True`)

```python
# settings.py
TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [BASE_DIR / "templates"],   # project-level templates
        "APP_DIRS": True,                   # also searches <app>/templates/
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",    # makes {{ user }} available in all templates
                "django.contrib.messages.context_processors.messages",  # makes {{ messages }} available
            ],
        },
    },
]
```

**Convention:** `<app>/templates/<app>/<template>.html` — the app name subfolder prevents collisions between apps. `wardrobe/templates/wardrobe/clothe_list.html` is referenced in code as `"wardrobe/clothe_list.html"`. Without the subfolder, two apps both named `clothe_list.html` would conflict.

[↑ Back to Contents](#contents)

---

## Context

Context is a dict passed from a view to a template. Its keys become template variables.

```python
# views.py
from django.shortcuts import render
from django.http import HttpRequest, HttpResponse
from .models import Clothe


def clothe_list(request: HttpRequest) -> HttpResponse:
    return render(request, "wardrobe/clothe_list.html", {
        "clothes": Clothe.objects.select_related("category").all(),
        "page_title": "My Wardrobe",
    })
```

```html
<!-- wardrobe/clothe_list.html -->
<h1>{{ page_title }}</h1>

{% for clothe in clothes %}
  <p>{{ clothe.type }} — {{ clothe.category.name }}</p>
{% endfor %}
```

Variable output syntax:

```html
{{ value }}                       <!-- output the value (HTML-escaped by default) -->
{{ value|lower }}                 <!-- apply a filter -->
{{ value|truncatechars:80 }}      <!-- filter with an argument -->
{{ clothe.category.name }}        <!-- dot notation traverses FK relationships -->
{{ clothe.get_type_display }}     <!-- calls model methods (no parentheses needed) -->
```

**Auto-escaping:** Django auto-escapes HTML in `{{ }}` — characters like `<`, `>`, `&`, `"` are converted to their HTML entities. This prevents XSS attacks. Use `|safe` or `format_html` deliberately when you need raw HTML output (and only for content you control).

### What's available in every template automatically

The context processors listed in `settings.py` inject variables into every template without you having to pass them manually from each view:

| Variable | From context processor | What it is |
| --- | --- | --- |
| `{{ user }}` | `auth` | The current user (or `AnonymousUser`) — see `06-django-auth.md` |
| `{{ messages }}` | `messages` | Flash messages from the messages framework — see `03-django-views-crud.md` |
| `{{ request }}` | `request` | The current `HttpRequest` object |
| `{{ debug }}` | `debug` | `True` if `DEBUG = True` in settings |

This is why you can write `{% if user.is_authenticated %}` in any template without passing `user` from the view — `django.contrib.auth.context_processors.auth` does that automatically.

[↑ Back to Contents](#contents)

---

## Context Processors

A context processor is a callable that receives `request` and returns a dict. That dict is merged into **every** template context automatically, so you don't have to pass the same data from every view.

```python
# wardrobe/context_processors.py
from django.http import HttpRequest
from .models import Category


def navigation(request: HttpRequest) -> dict:
    return {
        "nav_categories": Category.objects.filter(active=True).order_by("name"),
    }
```

Register it in settings:

```python
# settings.py
"context_processors": [
    "django.template.context_processors.request",
    "django.contrib.auth.context_processors.auth",
    "django.contrib.messages.context_processors.messages",
    "wardrobe.context_processors.navigation",   # ← add yours here
],
```

`nav_categories` is now available in every template — useful for navbars, sidebars, or any data that appears on every page.

> **Performance note:** context processors run on every request, for every view. Keep them lightweight — use `select_related` / `prefetch_related` and consider caching if the query is expensive.

[↑ Back to Contents](#contents)

---

## Template Inheritance

Template inheritance lets you define a base layout once and fill in named slots in child templates. This is how you avoid duplicating your `<head>`, `<nav>`, and `<footer>` across every page.

A base template defines named `{% block %}` slots. Child templates fill them in — anything not overridden falls back to the parent's default.

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>{% block title %}My Site{% endblock %}</title>
  {% load static %}
  <link rel="stylesheet" href="{% static 'css/main.css' %}" />
  {% block extra_css %}{% endblock %}
</head>
<body>
  <nav>{% block nav %}{% endblock %}</nav>

  <main>
    {% block content %}{% endblock %}
  </main>

  <footer>{% block footer %}{% endblock %}</footer>

  {% block extra_js %}{% endblock %}
</body>
</html>
```

Child template — override specific blocks, leave the rest as parent defaults:

```html
{% extends "base.html" %}

{% block title %}My Wardrobe{% endblock %}

{% block content %}
  <h1>Clothes</h1>
  {% for clothe in clothes %}
    <p>{{ clothe.type }}</p>
  {% endfor %}
{% endblock %}
```

### block.super

`{{ block.super }}` renders the parent block's content before adding your own. Use it when you want to extend a block rather than replace it entirely:

```html
<!-- templates/base.html -->
{% block extra_css %}
  <link rel="stylesheet" href="{% static 'css/base.css' %}" />
{% endblock %}

<!-- wardrobe/clothe_list.html -->
{% extends "base.html" %}

{% block extra_css %}
  {{ block.super }}  {# keeps base.css from the parent #}
  <link rel="stylesheet" href="{% static 'css/wardrobe.css' %}" />
{% endblock %}
```

Common uses: appending to a parent's CSS or JS block, extending a breadcrumb trail, adding page-specific scripts.

**Rules:**
- `{% extends %}` must be the **first tag** in the file.
- Blocks can be nested.
- For readability, name closing tags: `{% endblock content %}`.

[↑ Back to Contents](#contents)

---

## Built-in Tags

### url

Reverse a named URL without hardcoding the path. This is the template equivalent of `reverse()` in Python — if you rename a URL pattern, every `{% url %}` tag updates automatically:

```html
<a href="{% url 'wardrobe:clothe_list' %}">All clothes</a>
<a href="{% url 'wardrobe:clothe_detail' pk=clothe.pk %}">View</a>
<a href="{% url 'wardrobe:clothe_detail' clothe.pk %}">View (positional)</a>
```

Store the URL in a variable with `as`:

```html
{% url 'wardrobe:clothe_detail' clothe.pk as clothe_url %}
{% if clothe_url %}
  <a href="{{ clothe_url }}">View</a>
{% endif %}
```

The namespace (`wardrobe:`) comes from `app_name` in `urls.py` — see `03-django-views-crud.md`.

### static

Outputs the URL for a static file (CSS, JS, images). Requires `{% load static %}` at the top of the template:

```html
{% load static %}

<link rel="stylesheet" href="{% static 'css/main.css' %}" />
<script src="{% static 'js/app.js' %}"></script>
<img src="{% static 'images/logo.png' %}" alt="Logo" />
```

Store as a variable:

```html
{% static 'images/placeholder.png' as placeholder_url %}
<img src="{{ placeholder_url }}" alt="placeholder" />
```

### include

Renders a child template inline. Covered in detail in [Template Partials](#template-partials).

### if / elif / else

```html
{% if clothe.image %}
  <img src="{{ clothe.image.url }}" alt="preview" />
{% elif clothe.placeholder %}
  <img src="{{ clothe.placeholder.url }}" alt="placeholder" />
{% else %}
  <span>No image</span>
{% endif %}
```

Supported operators: `and`, `or`, `not`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `in`, `not in`, `is`, `is not`.

```html
{# Auth checks are common in templates — user is available via context processor #}
{% if user.is_authenticated %}
  Hello, {{ user.username }}!
{% else %}
  <a href="{% url 'login' %}">Log in</a>
{% endif %}

{% if perms.wardrobe.add_clothe %}
  <a href="{% url 'wardrobe:create' %}">Add item</a>
{% endif %}
```

### for

```html
{% for clothe in clothes %}
  <li>{{ forloop.counter }}. {{ clothe.type }}</li>
{% empty %}
  <li>No items found.</li>
{% endfor %}
```

`forloop` variables available inside a loop:

| Variable | Value |
| --- | --- |
| `forloop.counter` | 1-indexed iteration count |
| `forloop.counter0` | 0-indexed |
| `forloop.revcounter` | count from the end (1-indexed) |
| `forloop.first` | `True` on first iteration |
| `forloop.last` | `True` on last iteration |
| `forloop.length` | total number of items in the loop (Django 6.0+) |
| `forloop.parentloop` | reference to the enclosing loop (for nested loops) |

### with

Aliases a value for a block scope — avoids repeating expensive attribute lookups and keeps templates readable:

```html
{% with total=clothes.count %}
  <p>You have {{ total }} item{{ total|pluralize }}.</p>
{% endwith %}
```

Multiple aliases in one tag:

```html
{% with cat=clothe.category img=clothe.image %}
  <p>{{ cat.name }}: {% if img %}{{ img.url }}{% endif %}</p>
{% endwith %}
```

### Other Useful Tags

```html
{% comment %}
  This block is ignored by the template engine.
  Useful for temporarily disabling markup.
{% endcomment %}

{% verbatim %}
  {{ this won't be processed }} by Django.
  Useful when embedding JS template syntax (e.g. Vue, Alpine, Handlebars).
{% endverbatim %}

{% csrf_token %}
{# Required in every POST form — renders a hidden CSRF input. #}
{# Without it Django rejects the request with a 403 Forbidden. #}

{% load wardrobe_tags %}
{# Loads a custom tag/filter library. Must appear before any of its tags. #}
{# Load multiple at once: {% load wardrobe_tags humanize %} #}

{% now "Y-m-d" %}
{# Renders the current server date/time in the given strftime-style format. #}

{# Single-line comment — not rendered to HTML output #}
```

[↑ Back to Contents](#contents)

---

## Built-in Filters

Applied with `|`: `{{ value|filter }}` or `{{ value|filter:arg }}`.

```html
{# Strings #}
{{ clothe.description|truncatechars:100 }}     {# cut at N chars, add ellipsis #}
{{ clothe.description|truncatewords:20 }}      {# cut at N words #}
{{ name|lower }}
{{ name|upper }}
{{ name|title }}
{{ name|capfirst }}
{{ name|slugify }}
{{ value|default:"N/A" }}                      {# if value is falsy #}
{{ value|default_if_none:"N/A" }}              {# only if value is None #}

{# Numbers #}
{{ price|floatformat:2 }}                      {# "1500.00" #}
{{ price|floatformat:"-2" }}                   {# omit trailing zeros: "1500" or "1500.50" #}
{{ count|pluralize }}                          {# "" or "s" #}
{{ count|pluralize:"y,ies" }}                  {# "category" / "categories" #}

{# Dates #}
{{ clothe.created_at|date:"d M Y" }}           {# "01 Jan 2025" #}
{{ clothe.created_at|timesince }}              {# "3 days, 2 hours" ago #}
{{ clothe.created_at|timeuntil }}              {# time from now until value #}

{# Lists / QuerySets #}
{{ items|length }}
{{ items|first }}
{{ items|last }}
{{ tag_list|join:", " }}

{# HTML #}
{{ html_content|safe }}                        {# disable auto-escaping — only for trusted content #}
{{ value|linebreaks }}                         {# \n → <p> and <br> #}
{{ value|linebreaksbr }}                       {# \n → <br> only #}
{{ value|striptags }}                          {# strip all HTML tags #}
{{ value|escape }}                             {# force escape (e.g. inside |safe context) #}
{{ value|wordcount }}
{{ value|yesno:"yes,no,maybe" }}
```

[↑ Back to Contents](#contents)

---

## Custom Template Tags

Custom template tags let you move logic out of templates (which should stay dumb) and into Python, where it's testable and reusable.

### Setup

Create a `templatetags/` directory inside your app. It must be a Python package (needs an `__init__.py`):

```
wardrobe/
    templatetags/
        __init__.py
        wardrobe_tags.py
```

Every file in `templatetags/` is a loadable library. Load it at the top of a template before using its tags:

```html
{% load wardrobe_tags %}
```

### simple_tag

Returns a value directly into the template. Use for computed values and data lookups:

```python
# wardrobe/templatetags/wardrobe_tags.py
from django import template
from ..models import Clothe

register = template.Library()


@register.simple_tag
def total_clothes() -> int:
    return Clothe.objects.count()


@register.simple_tag
def clothes_in_category(category_slug: str) -> int:
    return Clothe.objects.filter(category__slug=category_slug).count()
```

```html
{% load wardrobe_tags %}

<p>Total items: {% total_clothes %}</p>
<p>Tops: {% clothes_in_category "tops" %}</p>
```

Store the result in a variable with `as`:

```html
{% total_clothes as wardrobe_count %}
<p>You have {{ wardrobe_count }} items.</p>
```

### inclusion_tag

Renders a sub-template and inserts the result. Ideal for reusable UI fragments — cards, sidebars, navbars. The function returns a context dict for the sub-template:

```python
from ..models import Clothe, Category


@register.inclusion_tag("wardrobe/tags/_clothe_card.html")
def clothe_card(clothe: Clothe, show_price: bool = False) -> dict:
    return {
        "clothe": clothe,
        "show_price": show_price,
    }


@register.inclusion_tag("wardrobe/tags/_category_nav.html")
def category_nav() -> dict:
    return {
        "categories": Category.objects.filter(active=True),
    }
```

```html
<!-- wardrobe/tags/_clothe_card.html -->
<div class="card">
  <h3>{{ clothe.type }}</h3>
  {% if clothe.image %}
    <img src="{{ clothe.image.url }}" alt="{{ clothe.type }}" />
  {% endif %}
  {% if show_price %}
    <p>{{ clothe.price }}</p>
  {% endif %}
</div>
```

```html
{% load wardrobe_tags %}

{% for clothe in clothes %}
  {% clothe_card clothe %}
  {% clothe_card clothe show_price=True %}
{% endfor %}
```

### Passing Context to Tags

Use `takes_context=True` to receive the full template context. The first parameter must be named `context`. Useful when the tag needs `request`, `user`, or any other context variable without explicit arguments:

```python
@register.simple_tag(takes_context=True)
def active_nav(context: dict, url_name: str) -> str:
    """Returns 'active' CSS class if the current URL matches url_name."""
    request = context["request"]
    from django.urls import reverse, NoReverseMatch
    try:
        return "active" if request.path == reverse(url_name) else ""
    except NoReverseMatch:
        return ""
```

```html
<li class="nav-item {% active_nav 'wardrobe:clothe_list' %}">
  <a href="{% url 'wardrobe:clothe_list' %}">Clothes</a>
</li>
```

Same for `inclusion_tag`:

```python
@register.inclusion_tag("wardrobe/tags/_user_badge.html", takes_context=True)
def user_badge(context: dict) -> dict:
    return {"user": context["request"].user}
```

### Custom Filter

```python
from django.utils.html import format_html


@register.filter(name="currency")
def currency(value: float, symbol: str = "KES") -> str:
    """{{ price|currency }} or {{ price|currency:"USD" }}"""
    try:
        return f"{symbol} {float(value):,.2f}"
    except (ValueError, TypeError):
        return str(value)


@register.filter(name="badge", is_safe=True)
def badge(value: str, css_class: str = "badge-primary") -> str:
    """Wrap a value in a Bootstrap badge span."""
    return format_html('<span class="badge {}">{}</span>', css_class, value)
```

```html
{% load wardrobe_tags %}

{{ clothe.price|currency }}              {# "KES 1,500.00" #}
{{ clothe.price|currency:"USD" }}        {# "USD 1,500.00" #}
{{ clothe.type|badge }}                  {# <span class="badge badge-primary">Shirt</span> #}
{{ clothe.type|badge:"badge-success" }}
```

- `is_safe=True` tells Django the output is already HTML-safe; auto-escaping won't double-escape it.
- Always use `format_html` (never string concatenation) when building HTML inside a tag or filter — it escapes any user-supplied values automatically.

[↑ Back to Contents](#contents)

---

## Template Partials

### include

Renders a child template inline. The current context is passed through automatically:

```html
{% include "wardrobe/partials/_clothe_card.html" %}
```

Pass extra variables with `with`:

```html
{% include "wardrobe/partials/_clothe_card.html" with show_price=True %}
```

Isolate the partial from the parent context using `only` — only `with` variables are passed:

```html
{% include "wardrobe/partials/_clothe_card.html" with clothe=featured only %}
```

Use a variable as the template name (dynamic includes):

```html
{% include template_name %}
```

**Convention:** prefix partial filenames with `_` to signal they are not standalone pages (e.g. `_clothe_card.html`, `_pagination.html`).

---

### Native Partials (Django 6.0+)

Django 6.0 added built-in template partials — named, reusable fragments defined inside a template file using `{% partialdef %}` and rendered with `{% partial %}`. No extra packages needed.

**Define a partial:**

```html
{# wardrobe/clothe_list.html #}

{% partialdef clothe-row %}
  <tr>
    <td>{{ clothe.type }}</td>
    <td>{{ clothe.category.name }}</td>
    <td>{{ clothe.price }}</td>
  </tr>
{% endpartialdef %}
{# or {% endpartialdef clothe-row %} for readability #}
```

**Render it elsewhere in the same template:**

```html
<table>
  <tbody>
    {% for clothe in clothes %}
      {% partial clothe-row %}
    {% endfor %}
  </tbody>
</table>
```

**Inline partials** — define and render at the same point, then reuse later:

```html
{# renders immediately AND makes the fragment available for reuse #}
{% partialdef clothe-row inline %}
  <tr>
    <td>{{ clothe.type }}</td>
    <td>{{ clothe.category.name }}</td>
  </tr>
{% endpartialdef %}

{# reuse it later in the same template #}
{% with clothe=featured_clothe %}
  {% partial clothe-row %}
{% endwith %}
```

**Target a partial from a view** — useful for HTMX responses that update only part of a page:

```python
# views.py
def clothe_list(request: HttpRequest) -> HttpResponse:
    clothes = Clothe.objects.all()
    template = "wardrobe/clothe_list.html"

    # HTMX hit — return only the table rows fragment
    if request.headers.get("HX-Request"):
        template += "#clothe-row"

    return render(request, template, {"clothes": clothes})
```

**Target a partial with `{% include %}`:**

```html
{% include "wardrobe/clothe_list.html#clothe-row" %}
```

**Context handling** — partials render with the current template context. Use `{% with %}` to supply specific variables:

```html
{% for clothe in clothes %}
  {% partial clothe-row %}              {# clothe is already in context #}
{% endfor %}

{% with clothe=featured_clothe %}
  {% partial clothe-row %}
{% endwith %}
```

> **`{% include %}` vs `{% partial %}` vs `inclusion_tag` — when to use which:**
>
> | Tool | Best for |
> | --- | --- |
> | `{% include %}` | Reusing a fragment that lives in its own file; sharing between templates |
> | `{% partial %}` / `{% partialdef %}` | Fragments defined in the same file; HTMX partial responses |
> | `inclusion_tag` | Fragments that need Python logic to build their own context (DB queries, computed values) |

[↑ Back to Contents](#contents)

---

## References

- https://docs.djangoproject.com/en/stable/topics/templates/
- https://docs.djangoproject.com/en/stable/ref/templates/builtins/
- https://docs.djangoproject.com/en/stable/howto/custom-template-tags/
- https://docs.djangoproject.com/en/stable/ref/templates/api/
- https://docs.djangoproject.com/en/6.0/releases/6.0/#template-partials
