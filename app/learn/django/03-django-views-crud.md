<!-- ramani: app/learn/django/03-django-views-crud.md -->
⬅️ [Back to Backend Notes](README.md)

---

# Django: Views & CRUD

## Contents

- [How Views Work](#how-views-work)
- [URL Configuration](#url-configuration)
- [Class-Based Views (CBVs)](#class-based-views-cbvs)
- [Full CRUD with CBVs](#full-crud-with-cbvs)
  - [List](#list)
  - [Detail](#detail)
  - [Create](#create)
  - [Update](#update)
  - [Delete](#delete)
- [Full CRUD with Generic CBVs](#full-crud-with-generic-cbvs)
  - [Default Template Names](#default-template-names)
  - [Views](#views)
- [Shortcuts Reference](#shortcuts-reference)
- [HttpRequest and HttpResponse](#httprequest-and-httpresponse)
- [Returning JSON](#returning-json)
- [Messages Framework](#messages-framework)
- [Protecting Views with Auth](#protecting-views-with-auth)
- [References](#references)

---

## How Views Work

A view is a callable that receives an `HttpRequest` and returns an `HttpResponse`. That's the entire contract.

```
Browser
  │
  ▼
URL dispatcher (urls.py)
  │  matches the URL pattern to a view function
  ▼
View (views.py)
  │  queries the DB via models, processes forms, builds context
  ▼
Template (*.html)
  │  renders HTML using context data
  ▼
HttpResponse
  │
  ▼
Browser
```

The view is where everything comes together. It imports from:

- **Models** (`01-django-models.md`) — to query the database (`Article.objects.filter(...)`)
- **Forms** (`02-django-forms.md`) — to validate POST data and create/update model instances
- **Templates** (`04-django-templates-tags.md`) — to render HTML with `render()`
- **Auth** (`06-django-auth.md`) — to restrict access with `@login_required` or `LoginRequiredMixin`

[↑ Back to Contents](#contents)

---

## URL Configuration

`urls.py` maps URL patterns to views. Django tries each pattern in order and calls the first matching view.

```python
# wardrobe/urls.py
from django.urls import path
from . import views

app_name = "wardrobe"   # enables namespacing: reverse("wardrobe:list")

urlpatterns = [
    path("", views.ClotheListView.as_view(), name="list"),
    path("<int:pk>/", views.ClotheDetailView.as_view(), name="detail"),
    path("new/", views.ClotheCreateView.as_view(), name="create"),
    path("<int:pk>/edit/", views.ClotheUpdateView.as_view(), name="update"),
    path("<int:pk>/delete/", views.ClotheDeleteView.as_view(), name="delete"),
]
```

Include app URLs in the project root:

```python
# project/urls.py
from django.urls import path, include

urlpatterns = [
    path("wardrobe/", include("wardrobe.urls")),
]
```

> **`app_name` vs Python package path — three independent things:**
>
> | Attribute | Where | Purpose | Example |
> | --- | --- | --- | --- |
> | `AppConfig.name` | `apps.py` | Python import path Django uses to find the app | `"app.wardrobe"` |
> | `AppConfig.label` | `apps.py` | Short identifier for migrations, admin, etc. | `"wardrobe"` (default) |
> | `app_name` | `urls.py` | URL namespace for `reverse()` and `{% url %}` | `"wardrobe"` |
>
> ```python
> # app/wardrobe/apps.py
> class WardrobeConfig(AppConfig):
>     name = "app.wardrobe"   # Python path — must match INSTALLED_APPS entry
>     label = "wardrobe"      # optional; already the default (last segment of name)
> ```
>
> ```python
> # app/wardrobe/urls.py
> app_name = "wardrobe"   # URL namespace — independent of the Python path above
> ```

Reverse a URL in Python:

```python
from django.urls import reverse
reverse("wardrobe:detail", kwargs={"pk": 42})
```

Reverse in a template:

```html
<a href="{% url 'wardrobe:detail' pk=clothe.pk %}">View</a>
```

[↑ Back to Contents](#contents)

---

## Class-Based Views (CBVs)

CBVs organise logic by HTTP method. The base `View` class dispatches incoming requests to methods named after the HTTP verb — `get()`, `post()`, `put()`, `delete()`. This avoids the `if request.method == "POST"` branching you'd write in a function-based view.

```python
from django.views import View
from django.shortcuts import render, redirect
from django.http import HttpRequest, HttpResponse

from .models import Clothe
from .forms import ClotheForm


class ClotheCreateView(View):
    def get(self, request: HttpRequest) -> HttpResponse:
        """Render the empty form."""
        form = ClotheForm()
        return render(request, "wardrobe/clothe_form.html", {"form": form})

    def post(self, request: HttpRequest) -> HttpResponse:
        """Process the submitted form."""
        form = ClotheForm(request.POST, request.FILES)
        if form.is_valid():
            form.save()
            return redirect("wardrobe:list")
        return render(request, "wardrobe/clothe_form.html", {"form": form})
```

Wire up with `.as_view()` in `urls.py`:

```python
path("new/", ClotheCreateView.as_view(), name="create"),
```

[↑ Back to Contents](#contents)

---

## Full CRUD with CBVs

### List

```python
from django.views import View
from django.shortcuts import render
from django.http import HttpRequest, HttpResponse
from .models import Clothe


class ClotheListView(View):
    def get(self, request: HttpRequest) -> HttpResponse:
        clothes = Clothe.objects.select_related("category").order_by("type")
        return render(request, "wardrobe/clothe_list.html", {"clothes": clothes})
```

Template (`wardrobe/clothe_list.html`):

```html
<ul>
  {% for clothe in clothes %}
  <li>
    {{ clothe.get_type_display }} — {{ clothe.category }}
    <a href="{% url 'wardrobe:detail' pk=clothe.pk %}">View</a>
    <a href="{% url 'wardrobe:update' pk=clothe.pk %}">Edit</a>
    <a href="{% url 'wardrobe:delete' pk=clothe.pk %}">Delete</a>
  </li>
  {% empty %}
  <li>No items yet.</li>
  {% endfor %}
</ul>
<a href="{% url 'wardrobe:create' %}">Add item</a>
```

`select_related("category")` fetches all clothes and their categories in one SQL JOIN, avoiding N+1 queries in the loop. See `01-django-models.md` → Reducing database queries.

### Detail

```python
from django.views import View
from django.shortcuts import render, get_object_or_404
from django.http import HttpRequest, HttpResponse
from .models import Clothe


class ClotheDetailView(View):
    def get_object(self, pk: int) -> Clothe:
        return get_object_or_404(Clothe, pk=pk)

    def get(self, request: HttpRequest, pk: int) -> HttpResponse:
        clothe = self.get_object(pk)
        return render(request, "wardrobe/clothe_detail.html", {"clothe": clothe})
```

Template (`wardrobe/clothe_detail.html`):

```html
<h1>{{ clothe.get_type_display }}</h1>
<p>Category: {{ clothe.category }}</p>
<p>{{ clothe.description }}</p>
{% if clothe.image %}
<img src="{{ clothe.image.url }}" alt="Image of {{ clothe.get_type_display }}" />
{% endif %}
<a href="{% url 'wardrobe:update' pk=clothe.pk %}">Edit</a>
<a href="{% url 'wardrobe:list' %}">Back to list</a>
```

### Create

```python
from django.views import View
from django.shortcuts import render, redirect
from django.http import HttpRequest, HttpResponse
from .forms import ClotheForm


class ClotheCreateView(View):
    def get(self, request: HttpRequest) -> HttpResponse:
        form = ClotheForm()
        return render(request, "wardrobe/clothe_form.html", {"form": form})

    def post(self, request: HttpRequest) -> HttpResponse:
        form = ClotheForm(request.POST, request.FILES)
        if form.is_valid():
            clothe = form.save()
            return redirect("wardrobe:detail", pk=clothe.pk)
        return render(request, "wardrobe/clothe_form.html", {"form": form})
```

Template (`wardrobe/clothe_form.html`):

```html
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}
  {% for field in form %}
  <div>{{ field.label_tag }} {{ field }} {{ field.errors }}</div>
  {% endfor %}
  <button type="submit">Save</button>
</form>
```

`enctype="multipart/form-data"` is required whenever the form has a `FileField` or `ImageField` — without it, uploaded files are not included in the POST body.

### Update

```python
from django.views import View
from django.shortcuts import render, redirect, get_object_or_404
from django.http import HttpRequest, HttpResponse
from .forms import ClotheForm
from .models import Clothe


class ClotheUpdateView(View):
    def get_object(self, pk: int) -> Clothe:
        return get_object_or_404(Clothe, pk=pk)

    def get(self, request: HttpRequest, pk: int) -> HttpResponse:
        clothe = self.get_object(pk)
        form = ClotheForm(instance=clothe)   # pre-populates the form
        return render(request, "wardrobe/clothe_form.html", {"form": form, "clothe": clothe})

    def post(self, request: HttpRequest, pk: int) -> HttpResponse:
        clothe = self.get_object(pk)
        form = ClotheForm(request.POST, request.FILES, instance=clothe)
        if form.is_valid():
            form.save()
            return redirect("wardrobe:detail", pk=clothe.pk)
        return render(request, "wardrobe/clothe_form.html", {"form": form, "clothe": clothe})
```

The same template used for create works for update — the form is pre-populated because of `instance=clothe`.

### Delete

```python
from django.views import View
from django.shortcuts import render, redirect, get_object_or_404
from django.http import HttpRequest, HttpResponse
from .models import Clothe


class ClotheDeleteView(View):
    def get_object(self, pk: int) -> Clothe:
        return get_object_or_404(Clothe, pk=pk)

    def get(self, request: HttpRequest, pk: int) -> HttpResponse:
        clothe = self.get_object(pk)
        return render(request, "wardrobe/clothe_confirm_delete.html", {"clothe": clothe})

    def post(self, request: HttpRequest, pk: int) -> HttpResponse:
        clothe = self.get_object(pk)
        clothe.delete()
        return redirect("wardrobe:list")
```

Template (`wardrobe/clothe_confirm_delete.html`):

```html
<p>Are you sure you want to delete "{{ clothe.get_type_display }}"?</p>
<form method="post">
  {% csrf_token %}
  <button type="submit">Yes, delete</button>
  <a href="{% url 'wardrobe:detail' pk=clothe.pk %}">Cancel</a>
</form>
```

> Always use a POST form for delete — never a bare `<a>` link. GET requests must not cause side effects. A link can be triggered by a browser prefetch, a bot, or an accidental click.

[↑ Back to Contents](#contents)

---

## Full CRUD with Generic CBVs

Django ships generic views for common patterns. They pre-write the boilerplate — less code, more convention. Under the hood they're still `View` subclasses; they just automate the queryset fetching, form binding, and redirect logic.

### Default Template Names

Each generic CBV resolves a default template based on the app label and model name. You only need `template_name` if you want to deviate from the convention.

| View | Default Template |
| --- | --- |
| `ListView` | `<app>/<model>_list.html` |
| `DetailView` | `<app>/<model>_detail.html` |
| `CreateView` | `<app>/<model>_form.html` |
| `UpdateView` | `<app>/<model>_form.html` |
| `DeleteView` | `<app>/<model>_confirm_delete.html` |

`CreateView` and `UpdateView` share the same default template — the form is pre-populated on update because the generic view passes `instance` automatically.

### Views

```python
from django.views.generic import (
    ListView,
    DetailView,
    CreateView,
    UpdateView,
    DeleteView,
)
from django.urls import reverse_lazy
from .models import Clothe
from .forms import ClotheForm


class ClotheListView(ListView):
    model = Clothe
    template_name = "wardrobe/clothe_list.html"
    context_object_name = "clothes"
    ordering = ["type"]

    def get_queryset(self):
        return super().get_queryset().select_related("category")


class ClotheDetailView(DetailView):
    model = Clothe
    template_name = "wardrobe/clothe_detail.html"
    context_object_name = "clothe"


class ClotheCreateView(CreateView):
    model = Clothe
    form_class = ClotheForm
    template_name = "wardrobe/clothe_form.html"

    def get_success_url(self):
        return reverse_lazy("wardrobe:detail", kwargs={"pk": self.object.pk})
        # If the model defines get_absolute_url(), you can omit get_success_url() entirely
        # and Django will redirect there automatically after a successful save.


class ClotheUpdateView(UpdateView):
    model = Clothe
    form_class = ClotheForm
    template_name = "wardrobe/clothe_form.html"

    def get_success_url(self):
        return reverse_lazy("wardrobe:detail", kwargs={"pk": self.object.pk})


class ClotheDeleteView(DeleteView):
    model = Clothe
    template_name = "wardrobe/clothe_confirm_delete.html"
    success_url = reverse_lazy("wardrobe:list")
```

> **`reverse_lazy` vs `reverse`:** `reverse_lazy` evaluates the URL pattern lazily — it's required at class level (like `success_url = reverse_lazy(...)`) because the URL configuration may not be loaded yet when Python imports the class. Inside a method (like `get_success_url`), either works, but `reverse_lazy` is conventional.

> **`get_absolute_url` shortcut:** If your model defines `get_absolute_url()` (see `01-django-models.md`), `CreateView` and `UpdateView` will redirect there automatically after a successful save — you can omit `get_success_url()` entirely.

Wire them up in `urls.py`:

```python
urlpatterns = [
    path("", ClotheListView.as_view(), name="list"),
    path("<int:pk>/", ClotheDetailView.as_view(), name="detail"),
    path("new/", ClotheCreateView.as_view(), name="create"),
    path("<int:pk>/edit/", ClotheUpdateView.as_view(), name="update"),
    path("<int:pk>/delete/", ClotheDeleteView.as_view(), name="delete"),
]
```

[↑ Back to Contents](#contents)

---

## Shortcuts Reference

```python
from django.shortcuts import render, redirect, get_object_or_404, get_list_or_404

# Render a template and return an HttpResponse
render(request, "template.html", {"key": value})

# Redirect to a named URL
redirect("wardrobe:list")
redirect("wardrobe:detail", pk=42)
redirect("/absolute/url/")

# Fetch one object, raise Http404 if not found — use in views instead of bare .get()
clothe = get_object_or_404(Clothe, pk=pk)
clothe = get_object_or_404(Clothe, slug=slug, published=True)

# Fetch a list, raise Http404 if the queryset is empty
clothes = get_list_or_404(Clothe, category=category)
```

`get_object_or_404` vs `Model.objects.get()`:

- `.get()` raises `DoesNotExist` — a Python exception that returns a 500 server error if uncaught
- `get_object_or_404` raises `Http404` — Django catches this and returns a proper 404 page

Always use `get_object_or_404` in views when looking up objects by user-supplied values (URL parameters, query strings).

[↑ Back to Contents](#contents)

---

## HttpRequest and HttpResponse

```python
def my_view(request: HttpRequest) -> HttpResponse:
    # Reading the request
    request.method          # 'GET', 'POST', 'PUT', 'DELETE', etc.
    request.GET             # query string params (?q=hello) — dict-like QueryDict
    request.POST            # form POST data — dict-like QueryDict
    request.FILES           # uploaded files
    request.user            # authenticated user (or AnonymousUser) — see 06-django-auth.md
    request.session         # session dict (persists across requests)
    request.headers         # HTTP headers (e.g. request.headers["Accept"])
    request.META            # full WSGI environment
    request.path            # '/wardrobe/42/'
    request.get_host()      # 'example.com'
    request.is_secure()     # True if HTTPS

    # Building responses
    return HttpResponse("Hello")
    return HttpResponse(content, status=201)
    return HttpResponseNotFound("Not found")       # 404
    return HttpResponseForbidden()                 # 403
    return HttpResponseRedirect("/somewhere/")     # 302
```

> `request.is_ajax()` was deprecated in Django 3.1 and removed in Django 4.0. To detect AJAX/fetch requests, check the `Accept` or `HX-Request` headers instead:
>
> ```python
> # Generic AJAX/fetch detection
> if request.headers.get("Accept") == "application/json":
>     return JsonResponse(data)
>
> # HTMX-specific
> if request.headers.get("HX-Request"):
>     return render(request, "wardrobe/clothe_list.html#clothe-row", context)
> ```

[↑ Back to Contents](#contents)

---

## Returning JSON

```python
from django.http import JsonResponse


def api_clothe_list(request: HttpRequest) -> JsonResponse:
    clothes = list(
        Clothe.objects.values("id", "type", "description")
    )
    return JsonResponse({"clothes": clothes})


# Non-dict response (list at the top level) — requires safe=False
return JsonResponse(clothes, safe=False)

# Custom status code
return JsonResponse({"error": "Not found"}, status=404)
```

`JsonResponse` automatically sets the `Content-Type: application/json` header and serializes the dict to JSON. It only accepts a dict by default — pass `safe=False` if the top-level value is a list.

[↑ Back to Contents](#contents)

---

## Messages Framework

Display one-time flash messages after a redirect. Because a redirect sends the browser to a new URL, you can't pass context data directly — the messages framework stores the message in the session and makes it available on the next request.

```python
from django.contrib import messages


class ClotheCreateView(View):
    def post(self, request: HttpRequest) -> HttpResponse:
        form = ClotheForm(request.POST, request.FILES)
        if form.is_valid():
            form.save()
            messages.success(request, "Item added to your wardrobe.")
            return redirect("wardrobe:list")
        return render(request, "wardrobe/clothe_form.html", {"form": form})
```

Display messages in the base template:

```html
{% if messages %}
<ul class="messages">
  {% for message in messages %}
  <li class="{{ message.tags }}">{{ message }}</li>
  {% endfor %}
</ul>
{% endif %}
```

Message levels (in order of severity):

```python
messages.debug(request, "...")
messages.info(request, "...")
messages.success(request, "...")
messages.warning(request, "...")
messages.error(request, "...")
```

### Tagging Messages

Pass `extra_tags` to attach custom tags — useful for driving different UI components (toasts vs inline alerts) from the same messages framework:

```python
messages.success(request, "Item saved!", extra_tags="toast")
messages.add_message(request, messages.SUCCESS, "Item saved!", extra_tags="toast")
```

`message.tags` in the template combines `extra_tags` and the built-in level tag:

```html
{% for message in messages %}
<div class="alert {{ message.tags }}">{{ message }}</div>
{% endfor %}
<!-- renders: class="alert toast success" -->
```

Access them separately if needed:

```python
message.tags        # "toast success"  (extra_tags + level tag combined)
message.extra_tags  # "toast"          (extra_tags only)
```

[↑ Back to Contents](#contents)

---

## Protecting Views with Auth

Views often need to be restricted to logged-in users, or to users with specific permissions. These patterns are covered fully in `06-django-auth.md` — here's a quick reference for where they slot into views.

### Function-based views — `@login_required`

```python
from django.contrib.auth.decorators import login_required


@login_required   # redirects to settings.LOGIN_URL if not authenticated
def clothe_create(request: HttpRequest) -> HttpResponse:
    ...
```

### Class-based views — `LoginRequiredMixin`

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import CreateView


class ClotheCreateView(LoginRequiredMixin, CreateView):
    model = Clothe
    form_class = ClotheForm
    # LoginRequiredMixin must come FIRST in the class definition (before CreateView)
```

### Permission checks

```python
from django.contrib.auth.decorators import permission_required
from django.contrib.auth.mixins import PermissionRequiredMixin


@permission_required("wardrobe.add_clothe", raise_exception=True)
def clothe_create(request):
    ...


class ClotheCreateView(LoginRequiredMixin, PermissionRequiredMixin, CreateView):
    permission_required = "wardrobe.add_clothe"
    raise_exception = True   # returns 403 instead of redirecting to login
```

### Attaching the logged-in user to a form save

A common pattern — the form doesn't include an `author` field (it would be a security risk to let users pick any author), so you set it after `commit=False`:

```python
class ArticleCreateView(LoginRequiredMixin, View):
    def post(self, request: HttpRequest) -> HttpResponse:
        form = ArticleForm(request.POST)
        if form.is_valid():
            article = form.save(commit=False)
            article.author = request.user   # set the logged-in user
            article.save()
            form.save_m2m()   # if the model has ManyToManyFields
            return redirect("articles:detail", pk=article.pk)
        return render(request, "articles/article_form.html", {"form": form})
```

### Role-based access (proxy model pattern)

If you're using the `Teacher`/`Pupil` proxy model pattern from `06-django-auth.md`, protect views by checking group membership:

```python
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin


class TeacherDashboardView(LoginRequiredMixin, UserPassesTestMixin, View):
    def test_func(self):
        return self.request.user.groups.filter(name="Teachers").exists()

    def get(self, request: HttpRequest) -> HttpResponse:
        ...
```

[↑ Back to Contents](#contents)

---

## References

- https://docs.djangoproject.com/en/stable/topics/http/views/
- https://docs.djangoproject.com/en/stable/topics/http/urls/
- https://docs.djangoproject.com/en/stable/ref/class-based-views/
- https://docs.djangoproject.com/en/stable/ref/request-response/
- https://docs.djangoproject.com/en/stable/ref/contrib/messages/
