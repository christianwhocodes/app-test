<!-- ramani: app/learn/django/02-django-forms.md -->
⬅️ [Back to Backend Notes](README.md)

---

# Django: Forms

## Contents

- [What is a Form?](#what-is-a-form)
- [Form vs ModelForm](#form-vs-modelform)
- [Defining a Form](#defining-a-form)
- [ModelForm](#modelform)
- [Common Field Types](#common-field-types)
- [Field Options](#field-options)
- [Widgets](#widgets)
- [Dynamic QuerySet Filtering](#dynamic-queryset-filtering)
- [Validation](#validation)
  - [Field-level Validation](#field-level-validation)
  - [Cross-field Validation](#cross-field-validation)
  - [How Form Validation Relates to Model Validation](#how-form-validation-relates-to-model-validation)
- [Using Forms in Views](#using-forms-in-views)
- [Rendering Forms in Templates](#rendering-forms-in-templates)
  - [What a field is made of](#what-a-field-is-made-of)
  - [Quick rendering (no styling control)](#quick-rendering-no-styling-control)
  - [Approach 1 — Style via attrs in Python](#approach-1--style-via-attrs-in-python)
  - [Approach 2 — Style via manual rendering in the template](#approach-2--style-via-manual-rendering-in-the-template)
  - [Summary table](#summary-table)
  - [Complete example](#complete-example)
  - [Loop rendering with styling](#loop-rendering-with-styling)
- [Forms in Auth](#forms-in-auth)
- [References](#references)

---

## What is a Form?

When a user fills in a field and clicks Submit, their browser sends that data to your server as raw text strings — regardless of whether they typed a number, a date, or an email address. A Django form is the layer that:

1. **Defines the expected fields** — what data do you want, and what type?
2. **Validates the input** — is it present? Is it in the right format? Does it meet your rules?
3. **Converts it to the right Python type** — a string `"42"` becomes an `int`, `"2024-01-15"` becomes a `date`
4. **Renders it as HTML** — draws `<input>`, `<select>`, `<textarea>` elements with labels and error messages

Without a form layer, you'd manually parse `request.POST`, write your own validation, handle errors, and build HTML by hand. Forms handle all of that for you.

```
Browser sends POST data (raw strings)
        ↓
   Django Form
        ↓
  form.is_valid()
     ↙        ↘
  True         False
   ↓              ↓
cleaned_data   form.errors
(Python types) (show errors to user)
```

[↑ Back to Contents](#contents)

---

## Form vs ModelForm

|  | `forms.Form` | `forms.ModelForm` |
| --- | --- | --- |
| **Use when** | Form not tied to a model | Form creates/updates a model instance |
| **Fields** | Defined manually | Auto-generated from the model |
| **Saving** | Handle manually | `.save()` method available |

If you're building a contact form, a search form, or a login form — reach for `forms.Form`. If you're creating or editing a model instance — reach for `forms.ModelForm`. It reads the model's field definitions so you don't define everything twice.

[↑ Back to Contents](#contents)

---

## Defining a Form

```python
from django import forms


class ContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)   # no forms.TextField — use CharField + Textarea widget
    subscribe = forms.BooleanField(required=False)
```

Each field performs type coercion and validation automatically. `forms.EmailField` validates that the submitted value looks like an email address. `forms.IntegerField` rejects non-numeric input.

[↑ Back to Contents](#contents)

---

## ModelForm

`ModelForm` generates fields automatically from a model. You only need to list which fields to include (or which to exclude). Django introspects the model and picks the right form field type for each — a `ForeignKey` becomes a `ModelChoiceField`, a `ManyToManyField` becomes `ModelMultipleChoiceField`, and so on.

```python
from django import forms
from .models import Clothe


class ClotheForm(forms.ModelForm):
    class Meta:
        model = Clothe
        fields = ["type", "description", "image", "category"]
        # or: exclude = ["created_at"]

        widgets = {
            "description": forms.Textarea(attrs={"rows": 3}),
        }
        labels = {
            "type": "Clothing type",
        }
        help_texts = {
            "image": "Optional. Upload a photo of the item.",
        }
```

### Saving a ModelForm

```python
form = ClotheForm(request.POST, request.FILES)   # include FILES for file/image uploads
if form.is_valid():
    clothe = form.save()   # creates a new row and returns the instance
```

### `commit=False` — do extra work before saving

Use `commit=False` when you need to set fields that aren't in the form (e.g. the logged-in user) before the row is written:

```python
clothe = form.save(commit=False)
clothe.added_by = request.user   # set a field not in the form
clothe.save()
form.save_m2m()   # REQUIRED when commit=False and the model has ManyToManyFields
                  # form.save() handles M2M automatically; commit=False skips that step
```

> `save_m2m()` must be called manually whenever you use `commit=False` and your model has any `ManyToManyField`. Django can't save M2M relations until the instance has a primary key — `commit=False` defers the `save()`, so M2M saving must also be deferred and called separately.

To see where `request` comes from and how forms fit into the request/response cycle, see [Using Forms in Views](#using-forms-in-views).

[↑ Back to Contents](#contents)

---

## Common Field Types

```python
name = forms.CharField(max_length=100)
email = forms.EmailField()
age = forms.IntegerField(min_value=0, max_value=150)
price = forms.DecimalField(max_digits=10, decimal_places=2)
start_date = forms.DateField(widget=forms.DateInput(attrs={"type": "date"}))
is_active = forms.BooleanField(required=False)
file = forms.FileField()
image = forms.ImageField()

# Static choices — hardcoded list of (value, label) tuples
status = forms.ChoiceField(choices=[("draft", "Draft"), ("published", "Published")])
tags = forms.MultipleChoiceField(choices=[("python", "Python"), ("django", "Django")])

# Dynamic choices — queryset from the database
category = forms.ModelChoiceField(queryset=Category.objects.all())
tags = forms.ModelMultipleChoiceField(queryset=Tag.objects.all())
```

### Choice Fields — ChoiceField vs ModelChoiceField

Use `ChoiceField`/`MultipleChoiceField` for fixed lists you define in code (status options, day names, categories that never change). Use `ModelChoiceField`/`ModelMultipleChoiceField` when the options come from database rows and can change at runtime.

### ModelForm auto-generates these fields for you

When using `ModelForm`, you rarely declare `ModelChoiceField` or `ModelMultipleChoiceField` manually. Django introspects the model and picks the right form field automatically:

| Model field | Auto-generated form field | Renders as |
| --- | --- | --- |
| `CharField`, `TextField` | `CharField` | `<input>` / `<textarea>` |
| `BooleanField` | `BooleanField` | `<input type="checkbox">` |
| `ForeignKey` | `ModelChoiceField` | `<select>` |
| `OneToOneField` | `ModelChoiceField` | `<select>` |
| `ManyToManyField` | `ModelMultipleChoiceField` | `<select multiple>` |

[↑ Back to Contents](#contents)

---

## Field Options

These are arguments you pass when declaring a field. They control the field's behaviour and how it renders:

```python
name = forms.CharField(
    max_length=100,
    required=True,       # default True; set False for optional fields
    label="Full name",   # overrides auto-generated label
    help_text="As on your ID.",
    initial="Anonymous", # pre-filled value shown on a blank form
    widget=forms.TextInput(
        attrs={"class": "form-control", "placeholder": "Enter name"}
    ),
    error_messages={
        "required": "Please provide your name.",
        "max_length": "Name is too long.",
    },
)
```

The `widget` option controls how the input element is rendered — its HTML tag and HTML attributes like `class`, `placeholder`, `rows`, `style`, or `data-*`. These attributes land directly on the `<input>` / `<textarea>` / `<select>` element.

What `widget` attrs **cannot** do is style the label, help text, or error messages — those are separate parts of a field that you control in the template. See [Rendering Forms in Templates](#rendering-forms-in-templates).

[↑ Back to Contents](#contents)

---

## Widgets

Widgets control the HTML rendered for a field. Each form field has a default widget — you only need to specify one when you want to override the default.

```python
class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = ["title", "body", "published_at", "status"]
        widgets = {
            "body": forms.Textarea(attrs={"rows": 10, "class": "editor"}),
            "published_at": forms.DateTimeInput(
                attrs={"type": "datetime-local"}, format="%Y-%m-%dT%H:%M"
            ),
            "status": forms.Select(attrs={"class": "form-select"}),
        }
```

Common widgets and their defaults:

| Widget | Renders as | Default for |
| --- | --- | --- |
| `TextInput` | `<input type="text">` | `CharField` |
| `Textarea` | `<textarea>` | — (use for `CharField` when you want multiline) |
| `EmailInput` | `<input type="email">` | `EmailField` |
| `PasswordInput` | `<input type="password">` | — |
| `NumberInput` | `<input type="number">` | `IntegerField`, `DecimalField` |
| `DateInput` | `<input type="date">` | `DateField` |
| `CheckboxInput` | `<input type="checkbox">` | `BooleanField` |
| `Select` | `<select>` | `ChoiceField`, `ModelChoiceField` |
| `SelectMultiple` | `<select multiple>` | `MultipleChoiceField`, `ModelMultipleChoiceField` |
| `HiddenInput` | `<input type="hidden">` | — |
| `FileInput` | `<input type="file">` | `FileField`, `ImageField` |

`ModelChoiceField` and `ModelMultipleChoiceField` render as `<select>` dropdowns by default — no widget declaration needed. Only specify a widget when overriding the default (e.g. swapping `Select` for `RadioSelect`).

> `attrs` on a widget only affect the rendered input element. The label, help text, and error messages are separate parts that you style in the template. See [Rendering Forms in Templates](#rendering-forms-in-templates).

[↑ Back to Contents](#contents)

---

## Dynamic QuerySet Filtering

Sometimes you need to restrict a `ModelChoiceField`'s options based on what's already been used — for example, showing only unbooked time slots. The filtering goes in `__init__`, not in the field declaration (because field declarations are evaluated once at class definition time, not per-request).

**The pattern:** `TimeSlot` holds the pool of available options. `Appointment` books one slot. Once a slot is booked, it no longer appears in the form.

```python
# models.py
class TimeSlot(models.Model):
    label = models.CharField(max_length=100)   # e.g. "Monday 9am–10am"
    date = models.DateField()
    start_time = models.TimeField()

    def __str__(self):
        return self.label

    class Meta:
        ordering = ["date", "start_time"]


class Appointment(models.Model):
    patient_name = models.CharField(max_length=200)
    slot = models.OneToOneField(   # OneToOne enforces at DB level that no slot is double-booked
        TimeSlot,
        on_delete=models.PROTECT,  # block deletion of a slot that has an appointment
        related_name="appointment",
    )
```

`OneToOneField` rather than `ForeignKey` here because a slot can only belong to one appointment — the DB enforces this. A plain `ForeignKey` would allow two appointments to reference the same slot.

```python
# forms.py
class AppointmentForm(forms.ModelForm):
    class Meta:
        model = Appointment
        fields = ["patient_name", "slot"]

    def __init__(self, *args, **kwargs):
        instance = kwargs.get("instance")
        super().__init__(*args, **kwargs)

        if instance and instance.pk:
            # On update: lock the field — reassigning an appointment to a different slot
            # isn't a legitimate edit. disabled=True greys it out and discards any
            # tampered POST value, always keeping the original slot.
            self.fields["slot"].disabled = True
        else:
            # On create: only show slots that have no appointment yet
            self.fields["slot"].queryset = TimeSlot.objects.filter(
                appointment__isnull=True
            ).order_by("date", "start_time")
```

`appointment__isnull=True` works because `related_name="appointment"` on the `OneToOneField` lets Django reverse the join: "give me slots that have no related Appointment row."

- **On create** — the queryset is filtered to only available slots.
- **On update** — `disabled=True` greys the field out in the UI and makes Django ignore whatever value is in the POST body, always falling back to the instance's existing value. A one-item queryset would look the same visually but could still be tampered with via a crafted POST request.

[↑ Back to Contents](#contents)

---

## Validation

### Field-level Validation

Add a `clean_<fieldname>` method to validate a single field. Raise `forms.ValidationError` for invalid input. **Return the cleaned value** at the end — Django replaces the field's value in `cleaned_data` with whatever you return.

```python
class RegistrationForm(forms.Form):
    username = forms.CharField(max_length=50)
    email = forms.EmailField()
    age = forms.IntegerField()

    def clean_username(self) -> str:
        username = self.cleaned_data["username"]
        if User.objects.filter(username=username).exists():
            raise forms.ValidationError("That username is already taken.")
        return username   # ← always return the value

    def clean_age(self) -> int:
        age = self.cleaned_data["age"]
        if age < 13:
            raise forms.ValidationError("You must be at least 13 years old.")
        return age
```

[↑ Back to Contents](#contents)

---

### Cross-field Validation

Override `clean()` to validate fields together. Always call `super().clean()` first — it runs all the individual field validators and populates `cleaned_data`. Use `.get()` rather than `[]` when reading from `cleaned_data` inside `clean()`, because a field that already failed its own validation won't be present in the dict.

#### Attaching errors to specific fields

Pass a dict to `ValidationError` to pin the error to a particular field. The form renders it next to that input rather than as a generic non-field error:

```python
class EventForm(forms.Form):
    start = forms.DateField()
    end = forms.DateField()

    def clean(self) -> dict:
        cleaned_data = super().clean()
        start = cleaned_data.get("start")
        end = cleaned_data.get("end")

        if start and end and end <= start:
            raise forms.ValidationError({"end": "End date must be after start date."})

        return cleaned_data
```

#### Collecting multiple errors with `add_error`

`self.add_error(field, error)` attaches an error to a named field and removes that field from `cleaned_data`. Pass `None` as the field name for a non-field error. This lets you collect all errors before returning rather than stopping at the first:

```python
def clean(self) -> dict:
    cleaned_data = super().clean()
    start = cleaned_data.get("start")
    end = cleaned_data.get("end")
    capacity = cleaned_data.get("capacity")

    if start and end and end <= start:
        self.add_error("end", "End date must be after start date.")

    if capacity is not None and capacity < 1:
        self.add_error("capacity", "Capacity must be at least 1.")

    return cleaned_data
```

[↑ Back to Contents](#contents)

---

### How Form Validation Relates to Model Validation

`forms.ValidationError` and `django.core.exceptions.ValidationError` are the same class — `forms` just re-exports it. The dict syntax for attaching errors to fields works identically in both:

| Where | Dict key | Error appears |
| --- | --- | --- |
| Model `clean()` | Field name | Next to that field in ModelForm |
| Form `clean()` | Field name | Next to that field in the rendered form |
| Either | `None` / plain string | As a non-field error above the form |

A `ModelForm` calls the model's `clean()` as part of `is_valid()`. This means cross-field validation you defined on the model (see `01-django-models.md`) surfaces automatically in any `ModelForm` for that model — you don't have to repeat it in the form.

The full validation order when `form.is_valid()` is called on a `ModelForm`:

1. `field.clean()` on each form field — type coercion + field-level validators
2. `form.clean_<fieldname>()` — your custom field-level methods
3. `form.clean()` — your form-level cross-field validation
4. Model's `full_clean()`:
   - `clean_fields()` — model field validators
   - `model.clean()` — model-level cross-field validation
   - `validate_unique()` — `unique` and `unique_together` constraints

**What this means in practice:** if you define a rule in the model's `clean()` (e.g. "end must be after start"), it fires automatically in the admin and in any `ModelForm` without you doing anything extra. If you define the rule only in the form's `clean()`, it won't fire in scripts or management commands that call `.save()` directly.

For the strongest guarantees, define the rule in both places — the model for correctness, the form for user-facing error messages — and add a `CheckConstraint` in `Meta` for absolute DB-level enforcement.

[↑ Back to Contents](#contents)

---

## Using Forms in Views

Use `django.views.View` to split GET and POST into separate methods. Each method receives the request and must return an `HttpResponse`. For a full explanation of views, see `03-django-views-crud.md`.

### Quick Reference

| Step | Create — GET | Create — POST | Update — GET | Update — POST |
| --- | --- | --- | --- | --- |
| **① Get obj** | — | — | `obj = self.get_object(pk)` | `obj = self.get_object(pk)` |
| **② Form** | `Form()` | `Form(POST, FILES)` | `Form(instance=obj)` | `Form(POST, FILES, instance=obj)` |
| **③ Return** | `render({form})` | `if valid → redirect`<br>`else → render({form})` | `render({form, obj})` | `if valid → redirect`<br>`else → render({form, obj})` |

```python
from django.views import View
from django.shortcuts import render, redirect, get_object_or_404
from django.http import HttpRequest, HttpResponse

from .models import Clothe
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


class ClotheUpdateView(View):
    def get_object(self, pk: int) -> Clothe:
        return get_object_or_404(Clothe, pk=pk)

    def get(self, request: HttpRequest, pk: int) -> HttpResponse:
        clothe = self.get_object(pk)
        form = ClotheForm(instance=clothe)   # pre-populates form with existing data
        return render(request, "wardrobe/clothe_form.html", {"form": form, "clothe": clothe})

    def post(self, request: HttpRequest, pk: int) -> HttpResponse:
        clothe = self.get_object(pk)
        form = ClotheForm(request.POST, request.FILES, instance=clothe)
        if form.is_valid():
            form.save()
            return redirect("wardrobe:detail", pk=clothe.pk)
        return render(request, "wardrobe/clothe_form.html", {"form": form, "clothe": clothe})
```

Wire them up in `urls.py` using `.as_view()`:

```python
from django.urls import path
from .views import ClotheCreateView, ClotheUpdateView

urlpatterns = [
    path("clothes/new/", ClotheCreateView.as_view(), name="clothe-create"),
    path("clothes/<int:pk>/edit/", ClotheUpdateView.as_view(), name="clothe-update"),
]
```

> **Why `get_object_or_404` appears in both methods on the update view:** GET and POST are separate method calls with no shared state. The object fetched in `get()` is gone by the time `post()` runs. This is fine for most views — each method is self-contained.

[↑ Back to Contents](#contents)

---

## Rendering Forms in Templates

When Django renders a form in a template, each field is not just one thing — it's four separate parts:

```
┌─────────────────────────────────────┐
│  Label          ← <label> element   │
│  Input          ← <input> element   │
│  Help text      ← plain text        │
│  Error messages ← <ul> of errors    │
└─────────────────────────────────────┘
```

You have template variables for each part, and you can render and style them independently.

---

### What a field is made of

For any field, say `form.name`, Django exposes these template variables:

| Variable | What it outputs |
| --- | --- |
| `{{ form.name }}` | The widget HTML — `<input>`, `<textarea>`, `<select>`, etc. |
| `{{ form.name.label_tag }}` | `<label for="id_name">Full name</label>` |
| `{{ form.name.label }}` | The label text only — `Full name` |
| `{{ form.name.id_for_label }}` | The `id` of the input — `id_name` |
| `{{ form.name.help_text }}` | The help text string |
| `{{ form.name.errors }}` | `<ul class="errorlist"><li>...</li></ul>` |
| `{{ form.non_field_errors }}` | Errors from `clean()` not tied to any field |

---

### Quick rendering (no styling control)

For prototyping, Django can render the whole form in one line:

```html
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  {# each field wrapped in <p> — also: form.as_table, form.as_ul #}
  <button type="submit">Save</button>
</form>
```

This gives you zero control over CSS classes, wrappers, or markup. Fine for a quick prototype, but you'll move away from it as soon as you need real styling.

---

### Approach 1 — Style via `attrs` in Python

`attrs` is a dict of HTML attributes added to the **widget element only** — the `<input>`, `<textarea>`, or `<select>`. It has no effect on the label, help text, or errors.

**On a plain `Form`:**

```python
class ContactForm(forms.Form):
    name = forms.CharField(
        widget=forms.TextInput(
            attrs={"class": "form-control", "placeholder": "Your full name"}
        )
    )
    message = forms.CharField(
        widget=forms.Textarea(attrs={"class": "form-control", "rows": 4})
    )
```

**On a `ModelForm`, inside `Meta.widgets`:**

```python
class ArticleForm(forms.ModelForm):
    class Meta:
        model = Article
        fields = ["title", "body"]
        widgets = {
            "title": forms.TextInput(attrs={"class": "form-control"}),
            "body": forms.Textarea(attrs={"class": "form-control", "rows": 6}),
        }
```

**What `attrs` controls and what it doesn't:**

| Part | Controlled by `attrs`? | Notes |
| --- | --- | --- |
| Input / widget | ✅ Yes | `class`, `placeholder`, `rows`, `style`, `data-*`, etc. |
| Label | ❌ No | Style in the template — see Approach 2 |
| Help text | ❌ No | Style in the template — see Approach 2 |
| Error messages | ❌ No | Style in the template — see Approach 2 |

---

### Approach 2 — Style via manual rendering in the template

When you render each part of a field separately, you can wrap each one in any HTML element and class you want.

#### Styling the label

`{{ field.label_tag }}` outputs `<label for="id_name">Full name</label>` with no class. To style the label, write it yourself:

```html
{# Built-in — no class, no control #}
{{ form.name.label_tag }}

{# Manual — add any class or attribute #}
<label for="{{ form.name.id_for_label }}" class="form-label fw-semibold">
  {{ form.name.label }}
</label>
```

`id_for_label` keeps the `for` attribute in sync with the input's `id`, so clicking the label still focuses the input.

#### Styling the input

`{{ form.name }}` renders the widget with whatever `attrs` you set in Python. The class has to be set in Python via `attrs` — you can't inject it from the template side:

```html
{# The class "form-control" was set in Python via attrs #}
{{ form.name }}
{# Output: <input type="text" name="name" class="form-control" id="id_name"> #}
```

#### Styling the help text

```html
{% if form.name.help_text %}
  <div class="form-text text-muted">{{ form.name.help_text }}</div>
{% endif %}
```

The `{% if %}` guard prevents an empty `<div>` from rendering when there's no help text.

#### Styling the errors

`{{ field.errors }}` outputs `<ul class="errorlist"><li>...</li></ul>`. You can target `.errorlist` with CSS, or loop the errors and build your own markup:

```html
{% for error in form.name.errors %}
  <div class="invalid-feedback d-block">{{ error }}</div>
{% endfor %}
```

For non-field errors (raised in `clean()` or via `add_error(None, ...)`):

```html
{% for error in form.non_field_errors %}
  <div class="alert alert-danger">{{ error }}</div>
{% endfor %}
```

---

### Summary table

| Part | Via `attrs` in Python | Via manual template rendering |
| --- | --- | --- |
| Input / widget | ✅ `attrs={"class": "form-control"}` | `{{ field }}` (inherits attrs from Python) |
| Label | ❌ Not possible | `<label for="{{ field.id_for_label }}" class="...">{{ field.label }}</label>` |
| Help text | ❌ Not possible | `<div class="...">{{ field.help_text }}</div>` |
| Errors | ❌ Not possible | Loop `field.errors`, wrap in your own HTML |

---

### Complete example

**Python — set input styling via `attrs`:**

```python
# forms.py
class ContactForm(forms.Form):
    name = forms.CharField(
        label="Full name",
        help_text="As it appears on your ID.",
        widget=forms.TextInput(
            attrs={"class": "form-control", "placeholder": "e.g. Jane Doe"}
        ),
        error_messages={"required": "Please enter your name."},
    )
```

**Template — style the label, help text, and errors manually:**

```html
<form method="post">
  {% csrf_token %}

  {% for error in form.non_field_errors %}
    <div class="alert alert-danger">{{ error }}</div>
  {% endfor %}

  <div class="mb-3">
    <label for="{{ form.name.id_for_label }}" class="form-label">
      {{ form.name.label }}
      {% if form.name.field.required %}<span class="text-danger">*</span>{% endif %}
    </label>

    {{ form.name }}

    {% if form.name.help_text %}
      <div class="form-text text-muted">{{ form.name.help_text }}</div>
    {% endif %}

    {% for error in form.name.errors %}
      <div class="invalid-feedback d-block">{{ error }}</div>
    {% endfor %}
  </div>

  <button type="submit" class="btn btn-primary">Submit</button>
</form>
```

---

### Loop rendering with styling

When you want consistent structure across all fields without writing each one manually:

```html
<form method="post" enctype="multipart/form-data">
  {% csrf_token %}

  {% for error in form.non_field_errors %}
    <div class="alert alert-danger">{{ error }}</div>
  {% endfor %}

  {% for field in form %}
    <div class="mb-3{% if field.errors %} has-error{% endif %}">
      <label for="{{ field.id_for_label }}" class="form-label">
        {{ field.label }}
        {% if field.field.required %}<span class="text-danger">*</span>{% endif %}
      </label>

      {{ field }}

      {% if field.help_text %}
        <div class="form-text text-muted">{{ field.help_text }}</div>
      {% endif %}

      {% for error in field.errors %}
        <div class="invalid-feedback d-block">{{ error }}</div>
      {% endfor %}
    </div>
  {% endfor %}

  <button type="submit" class="btn btn-primary">Save</button>
</form>
```

The trade-off with looping: all fields get the same wrapper markup. If you need different layouts for specific fields (a checkbox with its label inline, a hidden field with no wrapper, two fields side-by-side), switch to full manual rendering for those fields and use the loop for the rest.

> Use `enctype="multipart/form-data"` whenever the form has a `FileField` or `ImageField`. Without it, file data is not included in the POST body.
>
> Always include `{% csrf_token %}` in every form that sends a `POST` request. Django checks this token to confirm the request came from your own site. Without it, Django rejects the request with a 403.

[↑ Back to Contents](#contents)

---

## Forms in Auth

Django's authentication system (`06-django-auth.md`) ships with ready-made forms you can use directly or subclass:

| Form | Where to import | What it does |
| --- | --- | --- |
| `UserCreationForm` | `django.contrib.auth.forms` | Registration — `username`, `password1`, `password2` |
| `AuthenticationForm` | `django.contrib.auth.forms` | Login — used by Django's built-in `LoginView` |
| `PasswordChangeForm` | `django.contrib.auth.forms` | Changing password when logged in |
| `PasswordResetForm` | `django.contrib.auth.forms` | Sending a password reset email |
| `SetPasswordForm` | `django.contrib.auth.forms` | Setting a new password (used in reset flow) |

**Subclassing `UserCreationForm`** is the standard way to add extra fields (like email) to registration:

```python
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth import get_user_model

User = get_user_model()


class CustomRegistrationForm(UserCreationForm):
    email = forms.EmailField(required=True)

    class Meta(UserCreationForm.Meta):
        model = User
        fields = ["username", "email", "password1", "password2"]

    def clean_email(self) -> str:
        email = self.cleaned_data.get("email")
        if email and User.objects.filter(email=email).exists():
            raise forms.ValidationError("An account with this email already exists.")
        return email

    def save(self, commit: bool = True) -> User:
        user = super().save(commit=False)
        user.email = self.cleaned_data["email"]
        if commit:
            user.save()
        return user
```

The `clean_email` method is a field-level validator (covered in [Field-level Validation](#field-level-validation)) — `UserCreationForm` doesn't validate email uniqueness by default, so you add it here. Unless you had a custom user model, in which you overrode the email field and added a `unique=True` field option to it (see [01-django-models.md](./01-django-models.md) **Field Options**).

[↑ Back to Contents](#contents)

---

## References

- [Django Official Documentation: Forms](https://docs.djangoproject.com/en/stable/topics/forms/)
- [Django Official Documentation: ModelForms](https://docs.djangoproject.com/en/stable/topics/forms/modelforms/)
- [Django Form Fields Reference](https://docs.djangoproject.com/en/stable/ref/forms/fields/)
- [Django Widgets Reference](https://docs.djangoproject.com/en/stable/ref/forms/widgets/)
- [Django Form and Field Validation](https://docs.djangoproject.com/en/stable/ref/forms/validation/)
- [Django Built-in Auth Forms](https://docs.djangoproject.com/en/stable/topics/auth/default/#module-django.contrib.auth.forms)
