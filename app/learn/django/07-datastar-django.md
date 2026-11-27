<!-- ramani: app/learn/django/07-datastar-django.md -->
⬅️ [Back to Backend Notes](README.md)

---

# Datastar with Django

## Contents

- [What is Datastar?](#what-is-datastar)
- [How it Works](#how-it-works)
- [Setup](#setup)
- [Core Concepts](#core-concepts)
  - [Signals (`data-signals`)](#signals-data-signals)
  - [Binding (`data-bind`)](#binding-data-bind)
  - [Text (`data-text`)](#text-data-text)
  - [Fetching (`data-on` + `$$get` / `$$post`)](#fetching-data-on--get--post)
  - [Show and Hide (`data-show`)](#show-and-hide-data-show)
  - [Class Toggling (`data-class`)](#class-toggling-data-class)
- [Server-Sent Events (SSE)](#server-sent-events-sse)
  - [SSE Response Format](#sse-response-format)
  - [Django SSE View](#django-sse-view)
  - [StreamingHttpResponse Helper](#streaminghttpresponse-helper)
- [CRUD Patterns with Datastar](#crud-patterns-with-datastar)
  - [List with Live Search](#list-with-live-search)
  - [Inline Create](#inline-create)
  - [Inline Update](#inline-update)
  - [Inline Delete](#inline-delete)
- [CSRF with POST requests](#csrf-with-post-requests)
- [Tips and Gotchas](#tips-and-gotchas)
- [References](#references)

---

## What is Datastar?

Datastar is a lightweight (~14kb) JavaScript library that gives you reactive, hypermedia-driven UIs without writing JavaScript. You declare behaviour directly in HTML attributes and let the server stream HTML or JSON patches back via SSE.

Think of it as the middle ground between plain Django templates (full page reloads) and a full SPA (React/Vue with a separate API). You get SPA-like interactivity while keeping your logic in Python and your views server-rendered.

[↑ Back to Contents](#contents)

---

## How it Works

```
User interaction (click, input, submit)
    ↓
Datastar reads data-* attributes and current signal state
    ↓
Sends request to Django view (GET or POST via fetch)
    ↓
Django view returns an SSE stream
    ↓
Datastar reads SSE events and patches the DOM / updates signals
    ↓
Page updates without a full reload
```

[↑ Back to Contents](#contents)

---

## Setup

Add Datastar to your base template via CDN:

```html
<!-- base.html -->
<head>
  <script
    type="module"
    src="https://cdn.jsdelivr.net/gh/starfederation/datastar@v1.0.0-beta.11/bundles/datastar.js"
  ></script>
</head>
```

Or self-host by downloading the bundle and placing it in `static/js/`.

No npm, no build step required.

[↑ Back to Contents](#contents)

---

## Core Concepts

### Signals (`data-signals`)

Signals are reactive variables. Define them on any element — they're available to all child elements.

```html
<div data-signals="{query: '', count: 0, loading: false}">...</div>
```

Signals are like a client-side store scoped to the DOM subtree. When a signal changes, any element referencing it updates automatically.

### Binding (`data-bind`)

Two-way bind a signal to an input:

```html
<input type="text" data-bind-query placeholder="Search..." />
<!-- typing into this input updates the 'query' signal -->
```

One-way bind (signal → attribute):

```html
<input type="text" data-attr-value="query" />
```

### Text (`data-text`)

Render a signal's value as text content:

```html
<span data-text="count"></span> <span data-text="`Results: ${count}`"></span>
```

### Fetching (`data-on` + `$$get` / `$$post`)

Trigger a request on an event. Datastar includes the current signals as query params (GET) or JSON body (POST):

```html
<!-- Fire a GET on input -->
<input type="text" data-bind-query data-on-input="$$get('/wardrobe/search/')" />

<!-- Fire a POST on click -->
<button data-on-click="$$post('/wardrobe/items/')">Add item</button>

<!-- Debounce (wait 300ms after last keystroke) -->
<input
  data-bind-query
  data-on-input__debounce_300ms="$$get('/wardrobe/search/')"
/>
```

The view receives signals as query parameters or JSON. The response is an SSE stream that patches the DOM.

### Show and Hide (`data-show`)

```html
<div data-show="loading">Loading...</div>
<div data-show="!loading && results.length === 0">No results found.</div>
```

### Class Toggling (`data-class`)

```html
<button data-class="{active: isActive, disabled: loading}">Submit</button>
```

[↑ Back to Contents](#contents)

---

## Server-Sent Events (SSE)

Datastar communicates via SSE. Your Django views stream SSE events instead of returning a full HTML page.

### SSE Response Format

Two event types are most commonly used:

**`datastar-merge-fragments`** — patch HTML into the DOM:

```
event: datastar-merge-fragments
data: selector #my-element
data: mergeMode outer
data:
data: <div id="my-element">...new HTML...</div>
```

**`datastar-merge-signals`** — update signal values on the client:

```
event: datastar-merge-signals
data: signals {loading: false, count: 3}
```

Each event block ends with a blank line. The `selector` tells Datastar which element to target. `mergeMode` can be `outer` (replace the element), `inner` (replace children), `append`, `prepend`, etc.

### Django SSE View

```python
import json
from django.http import StreamingHttpResponse, HttpRequest
from django.template.loader import render_to_string
from .models import Clothe


def sse_clothe_search(request: HttpRequest) -> StreamingHttpResponse:
    query = request.GET.get("query", "").strip()

    def event_stream():
        clothes = Clothe.objects.filter(
            description__icontains=query
        ).select_related("category")

        # 1. Signal loading: false (search is done)
        yield "event: datastar-merge-signals\n"
        yield f"data: signals {json.dumps({'loading': False, 'count': clothes.count()})}\n"
        yield "\n"

        # 2. Patch the results HTML into the DOM
        html = render_to_string(
            "wardrobe/_clothe_list.html",
            {"clothes": clothes},
            request=request,
        )
        yield "event: datastar-merge-fragments\n"
        yield "data: selector #clothe-results\n"
        yield "data: mergeMode inner\n"
        yield "data:\n"
        yield f"data: {html}\n"
        yield "\n"

    return StreamingHttpResponse(
        event_stream(),
        content_type="text/event-stream",
    )
```

### StreamingHttpResponse Helper

A small helper keeps SSE formatting tidy:

```python
from typing import Generator


def sse_merge_fragments(selector: str, html: str, merge_mode: str = "outer") -> str:
    """Return a formatted SSE datastar-merge-fragments event string."""
    lines = [
        "event: datastar-merge-fragments",
        f"data: selector {selector}",
        f"data: mergeMode {merge_mode}",
        "data:",
        f"data: {html}",
        "",
        "",
    ]
    return "\n".join(lines)


def sse_merge_signals(signals: dict) -> str:
    """Return a formatted SSE datastar-merge-signals event string."""
    return f"event: datastar-merge-signals\ndata: signals {json.dumps(signals)}\n\n"
```

Use them in your views:

```python
def event_stream() -> Generator[str, None, None]:
    yield sse_merge_signals({"loading": False})
    yield sse_merge_fragments("#clothe-results", html, merge_mode="inner")
```

[↑ Back to Contents](#contents)

---

## CRUD Patterns with Datastar

### List with Live Search

Template:

```html
<div data-signals="{query: '', loading: false, count: 0}">
  <input
    type="text"
    data-bind-query
    data-on-input__debounce_300ms="$loading = true; $$get('/wardrobe/search/')"
    placeholder="Search your wardrobe..."
  />

  <span data-text="`${count} items`"></span>
  <div data-show="loading">Searching...</div>

  <div id="clothe-results">{% include "wardrobe/_clothe_list.html" %}</div>
</div>
```

View:

```python
def sse_clothe_search(request: HttpRequest) -> StreamingHttpResponse:
    query = request.GET.get("query", "").strip()
    clothes = Clothe.objects.filter(description__icontains=query).select_related("category")

    def stream():
        html = render_to_string("wardrobe/_clothe_list.html", {"clothes": clothes}, request=request)
        yield sse_merge_signals({"loading": False, "count": clothes.count()})
        yield sse_merge_fragments("#clothe-results", html, merge_mode="inner")

    return StreamingHttpResponse(stream(), content_type="text/event-stream")
```

Partial template (`wardrobe/_clothe_list.html`):

```html
{% for clothe in clothes %}
<div class="clothe-item" id="clothe-{{ clothe.pk }}">
  <span>{{ clothe.get_type_display }}</span>
  <span>{{ clothe.category }}</span>
  <button data-on-click="$$get('/wardrobe/{{ clothe.pk }}/edit-form/')">
    Edit
  </button>
  <button data-on-click="$$delete('/wardrobe/{{ clothe.pk }}/delete/')">
    Delete
  </button>
</div>
{% empty %}
<p>No items found.</p>
{% endfor %}
```

### Inline Create

Template — a form that posts without a page reload:

```html
<div id="clothe-create-form">
  <form data-on-submit="$$post('/wardrobe/create/')">
    {% csrf_token %} {{ form.as_p }}
    <button type="submit">Add item</button>
  </form>
</div>

<div id="clothe-results">{% include "wardrobe/_clothe_list.html" %}</div>
```

View:

```python
from django.views.decorators.http import require_POST


@require_POST
def sse_clothe_create(request: HttpRequest) -> StreamingHttpResponse:
    form = ClotheForm(request.POST, request.FILES)

    def stream():
        if form.is_valid():
            form.save()
            clothes = Clothe.objects.select_related("category").all()
            list_html = render_to_string("wardrobe/_clothe_list.html", {"clothes": clothes}, request=request)
            form_html = render_to_string("wardrobe/_clothe_form.html", {"form": ClotheForm()}, request=request)

            yield sse_merge_fragments("#clothe-results", list_html, merge_mode="inner")
            yield sse_merge_fragments("#clothe-create-form", form_html, merge_mode="inner")
        else:
            # Re-render the form with errors
            form_html = render_to_string("wardrobe/_clothe_form.html", {"form": form}, request=request)
            yield sse_merge_fragments("#clothe-create-form", form_html, merge_mode="inner")

    return StreamingHttpResponse(stream(), content_type="text/event-stream")
```

### Inline Update

Clicking "Edit" fetches an edit form fragment and swaps it into the item's row:

```python
def sse_clothe_edit_form(request: HttpRequest, pk: int) -> StreamingHttpResponse:
    clothe = get_object_or_404(Clothe, pk=pk)
    form = ClotheForm(instance=clothe)

    def stream():
        html = render_to_string(
            "wardrobe/_clothe_edit_form.html",
            {"form": form, "clothe": clothe},
            request=request,
        )
        yield sse_merge_fragments(f"#clothe-{pk}", html, merge_mode="outer")

    return StreamingHttpResponse(stream(), content_type="text/event-stream")


@require_POST
def sse_clothe_update(request: HttpRequest, pk: int) -> StreamingHttpResponse:
    clothe = get_object_or_404(Clothe, pk=pk)
    form = ClotheForm(request.POST, request.FILES, instance=clothe)

    def stream():
        if form.is_valid():
            form.save()
            html = render_to_string(
                "wardrobe/_clothe_item.html",
                {"clothe": clothe},
                request=request,
            )
            yield sse_merge_fragments(f"#clothe-{pk}", html, merge_mode="outer")
        else:
            html = render_to_string(
                "wardrobe/_clothe_edit_form.html",
                {"form": form, "clothe": clothe},
                request=request,
            )
            yield sse_merge_fragments(f"#clothe-{pk}", html, merge_mode="outer")

    return StreamingHttpResponse(stream(), content_type="text/event-stream")
```

Edit form partial (`wardrobe/_clothe_edit_form.html`):

```html
<div id="clothe-{{ clothe.pk }}">
  <form data-on-submit="$$post('/wardrobe/{{ clothe.pk }}/update/')">
    {% csrf_token %} {{ form.as_p }}
    <button type="submit">Save</button>
    <button
      type="button"
      data-on-click="$$get('/wardrobe/{{ clothe.pk }}/cancel/')"
    >
      Cancel
    </button>
  </form>
</div>
```

### Inline Delete

```python
@require_POST
def sse_clothe_delete(request: HttpRequest, pk: int) -> StreamingHttpResponse:
    clothe = get_object_or_404(Clothe, pk=pk)

    def stream():
        clothe.delete()
        # Remove the element from the DOM by replacing it with nothing
        yield sse_merge_fragments(f"#clothe-{pk}", "", merge_mode="outer")

    return StreamingHttpResponse(stream(), content_type="text/event-stream")
```

Trigger from the list partial:

```html
<button
  data-on-click="$$post('/wardrobe/{{ clothe.pk }}/delete/')"
  data-confirm="Are you sure?"
>
  Delete
</button>
```

[↑ Back to Contents](#contents)

---

## CSRF with POST requests

Datastar does not automatically include the CSRF token for POST requests. You need to include it in the request.

**Option 1 — Include the token as a signal:**

```html
<div data-signals="{csrftoken: '{{ csrf_token }}'}">
  <form data-on-submit="$$post('/wardrobe/create/')">...</form>
</div>
```

Datastar will include all current signals in the POST body, so the Django view receives `csrftoken` and you need to handle it — this is non-standard.

**Option 2 — Use a hidden input inside the form (simplest):**

```html
<form data-on-submit="$$post('/wardrobe/create/')">{% csrf_token %} ...</form>
```

Datastar serialises form fields when using `$$post` with a form's submit event, including the hidden `csrfmiddlewaretoken` input.

**Option 3 — Set the header globally via a meta tag (cleanest for many forms):**

```html
<!-- base.html -->
<meta name="csrf-token" content="{{ csrf_token }}" />
<script>
  document.addEventListener("datastar:fetch", (e) => {
    e.detail.headers["X-CSRFToken"] = document.querySelector(
      'meta[name="csrf-token"]',
    ).content;
  });
</script>
```

Then in `settings.py`, make sure `CSRF_TRUSTED_ORIGINS` and the `X-CSRFToken` header is accepted (it is by default in Django).

[↑ Back to Contents](#contents)

---

## Tips and Gotchas

**Fragment IDs must be stable** — the `selector` in your SSE event must match an `id` on an element that exists in the DOM at the time the response arrives. If the element is missing, Datastar silently does nothing.

**Always stream at least one event** — `StreamingHttpResponse` needs at least one `yield` before the connection closes, otherwise some browsers will retry immediately.

**Use partials, not full pages** — render the smallest possible HTML fragment in your SSE views. Keep full-page templates separate from partial (`_`) templates.

**Middleware ordering** — SSE responses are long-lived. If you have middleware that buffers responses (e.g. GZip middleware), it will prevent SSE from streaming. Disable buffering for SSE endpoints or exclude them from such middleware.

**`render_to_string` vs `render`** — inside a streaming generator you can't use `render()` (it returns an `HttpResponse`). Always use `render_to_string()` and yield the string.

**Content-Type** — always set `content_type="text/event-stream"` on `StreamingHttpResponse`. Without it, browsers will not treat the response as SSE.

**Multi-line HTML in SSE data** — each `data:` line in SSE is a separate chunk. If your rendered HTML contains newlines, you must either strip them or prefix each line with `data: `. The helper functions above handle this by rendering to a single line:

```python
def sse_merge_fragments(selector: str, html: str, merge_mode: str = "outer") -> str:
    # Collapse newlines so the HTML fits on one data: line
    html_single_line = html.replace("\n", " ").strip()
    return (
        "event: datastar-merge-fragments\n"
        f"data: selector {selector}\n"
        f"data: mergeMode {merge_mode}\n"
        "data:\n"
        f"data: {html_single_line}\n"
        "\n"
    )
```

[↑ Back to Contents](#contents)

---

## References

- https://data-star.dev/
- https://data-star.dev/guide/getting_started
- https://data-star.dev/reference/attribute_plugins
- https://docs.djangoproject.com/en/stable/ref/request-response/#streaminghttpresponse-objects
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
