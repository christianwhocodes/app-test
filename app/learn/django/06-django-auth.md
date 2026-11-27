<!-- ramani: app/learn/django/06-django-auth.md -->
⬅️ [Back to Backend Notes](README.md)

---

# Django: Authentication

## Contents

- [How Django Auth Works](#how-django-auth-works)
- [The User Model](#the-user-model)
- [Custom User Model](#custom-user-model)
  - [USERNAME\_FIELD and REQUIRED\_FIELDS](#username_field-and-required_fields)
- [Login and Logout](#login-and-logout)
  - [Using Django's Built-in Views](#using-djangos-built-in-views)
  - [Custom Login View](#custom-login-view)
- [Registration](#registration)
- [Protecting Views](#protecting-views)
  - [login\_required and method\_decorator](#login_required-and-method_decorator)
  - [LoginRequiredMixin](#loginrequiredmixin)
  - [Checking Manually](#checking-manually)
- [Permissions](#permissions)
  - [Built-in Permissions](#built-in-permissions)
  - [permission\_required and method\_decorator](#permission_required-and-method_decorator)
  - [PermissionRequiredMixin](#permissionrequiredmixin)
  - [Custom Permissions](#custom-permissions)
- [The request.user Object](#the-requestuser-object)
- [Password Management](#password-management)
- [Role-Based Users with Proxy Models](#role-based-users-with-proxy-models)
  - [The Problem This Solves](#the-problem-this-solves)
  - [How Proxy Models Work](#how-proxy-models-work)
  - [Step 1 — Define Roles with TextChoices](#step-1--define-roles-with-textchoices)
  - [Step 2 — Create the Groups via post\_migrate Signal](#step-2--create-the-groups-via-post_migrate-signal)
  - [Step 3 — Define the Proxy Models](#step-3--define-the-proxy-models)
  - [Step 4 — Proxy Model Managers](#step-4--proxy-model-managers)
  - [Step 5 — Profile Models](#step-5--profile-models)
  - [Step 6 — Forms](#step-6--forms)
  - [Step 7 — Views](#step-7--views)
  - [Step 8 — Admin](#step-8--admin)
  - [Step 9 — Checking Roles in Code](#step-9--checking-roles-in-code)
  - [What to Watch Out For](#what-to-watch-out-for)
- [References](#references)

---

## How Django Auth Works

Django's authentication system handles:

- **Authentication** — who is this user? (login)
- **Authorisation** — what can they do? (permissions)

It stores session data in `request.session` and the authenticated user in `request.user`. The session ID is kept in a cookie; the session data lives in the DB.

[↑ Back to Contents](#contents)

---

## The User Model

Django ships with `django.contrib.auth.models.User`. Always reference it via `get_user_model()` so your code works with custom user models too:

```python
from django.contrib.auth import get_user_model

User = get_user_model()
```

Default fields on `User`:

```python
user.username
user.email
user.first_name
user.last_name
user.password          # hashed; never access raw
user.is_active         # False disables login
user.is_staff          # can log in to admin
user.is_superuser      # all permissions
user.date_joined
user.last_login
```

[↑ Back to Contents](#contents)

---

## Custom User Model

**Set this up before the first migration.** It is very painful to change later.

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    """Extended user model with extra fields."""
    bio    = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True, null=True)
```

```python
# settings.py
AUTH_USER_MODEL = "accounts.User"
```

```python
# admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .models import User


@admin.register(User)
class CustomUserAdmin(UserAdmin):
    fieldsets = UserAdmin.fieldsets + (
        ("Profile", {"fields": ["bio", "avatar"]}),
    )
```

Then always import via `get_user_model()` everywhere else.

---

### USERNAME_FIELD and REQUIRED_FIELDS

Both are class attributes inherited from `AbstractBaseUser` (which `AbstractUser` extends).

| Attribute | Purpose | Default |
|-----------|---------|---------|
| `USERNAME_FIELD` | The field used to identify a user during login (passed to `authenticate(username=...)`) | `"username"` |
| `REQUIRED_FIELDS` | Extra fields prompted when running `createsuperuser` via the CLI | `["email"]` |

**Setting email as the login identifier:**

```python
class User(AbstractUser):
    email = models.EmailField(unique=True)   # must be unique when used as USERNAME_FIELD
    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ["username"]           # username still prompted by createsuperuser
```

**Does the `username` field disappear when you set `USERNAME_FIELD = "email"`?**

No. `AbstractUser` defines `username` as a concrete model field — it will still exist in the database. Setting `USERNAME_FIELD = "email"` changes which field is used as the login identifier, but does **not** remove the `username` column. If you genuinely want to drop `username` entirely, extend `AbstractBaseUser` (plus `PermissionsMixin`) instead of `AbstractUser` and define all fields yourself.

**Does `UserCreationForm` automatically validate email when `USERNAME_FIELD = "email"`?**

`UserCreationForm.Meta.fields` defaults to `("username",)`. Setting `USERNAME_FIELD = "email"` does **not** automatically change the form. If you want the form to use email, you must explicitly update `Meta.fields` when subclassing:

```python
class CustomRegistrationForm(UserCreationForm):
    class Meta(UserCreationForm.Meta):
        model = User
        fields = ["email", "password1", "password2"]   # username dropped, email promoted
```

**Does `UserCreationForm` include `REQUIRED_FIELDS`?**

No. `REQUIRED_FIELDS` only governs which fields the `createsuperuser` management command prompts for interactively. It has no effect on `UserCreationForm` at all. Add any extra fields to `Meta.fields` manually.

**Is there a `UserChangeForm`?**

Yes: `django.contrib.auth.forms.UserChangeForm`. It renders all user fields and is used internally by `UserAdmin` for the edit-user page in the Django admin. You can subclass it for an "edit profile" view:

```python
from django.contrib.auth.forms import UserChangeForm


class ProfileEditForm(UserChangeForm):
    class Meta(UserChangeForm.Meta):
        model = User
        fields = ["first_name", "last_name", "email"]
```

> **Note:** `UserChangeForm` includes a read-only password field (shows the hash with a link to the password-change view). Omit `"password"` from `fields` if you don't want it showing up.

[↑ Back to Contents](#contents)

---

## Login and Logout

### Using Django's Built-in Views

The quickest approach — wire up the built-in views and provide templates:

```python
# urls.py
from django.contrib.auth import views as auth_views
from django.urls import path

urlpatterns = [
    path("login/",  auth_views.LoginView.as_view(template_name="accounts/login.html"), name="login"),
    path("logout/", auth_views.LogoutView.as_view(), name="logout"),
]
```

```python
# settings.py
LOGIN_URL = "/login/"           # where to redirect unauthenticated users
LOGIN_REDIRECT_URL = "/"        # where to go after login
LOGOUT_REDIRECT_URL = "/login/" # where to go after logout
```

Template (`accounts/login.html`):

```html
<form method="post">
  {% csrf_token %} {{ form.as_p }}
  <button type="submit">Log in</button>
</form>
```

### Custom Login View

When you need more control:

```python
from django.contrib.auth import authenticate, login
from django.shortcuts import render, redirect
from django.http import HttpRequest, HttpResponse


def login_view(request: HttpRequest) -> HttpResponse:
    if request.user.is_authenticated:
        return redirect("home")

    error = None

    if request.method == "POST":
        username = request.POST.get("username", "").strip()
        password = request.POST.get("password", "")

        user = authenticate(request, username=username, password=password)
        if user is not None:
            login(request, user)
            next_url = request.GET.get("next") or "home"
            return redirect(next_url)
        else:
            error = "Invalid username or password."

    return render(request, "accounts/login.html", {"error": error})


def logout_view(request: HttpRequest) -> HttpResponse:
    from django.contrib.auth import logout
    logout(request)
    return redirect("login")
```

[↑ Back to Contents](#contents)

---

## Registration

```python
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth import login
from django.shortcuts import render, redirect
from django.http import HttpRequest, HttpResponse


def register_view(request: HttpRequest) -> HttpResponse:
    if request.user.is_authenticated:
        return redirect("home")

    if request.method == "POST":
        form = UserCreationForm(request.POST)
        if form.is_valid():
            user = form.save()
            login(request, user)   # log in immediately after registration
            return redirect("home")
    else:
        form = UserCreationForm()

    return render(request, "accounts/register.html", {"form": form})
```

For a custom registration form (e.g. to include email):

```python
from django import forms
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth import get_user_model

User = get_user_model()


class CustomRegistrationForm(UserCreationForm):
    email = forms.EmailField(required=True)

    class Meta(UserCreationForm.Meta):
        model = User
        fields = ["username", "email", "password1", "password2"]

    def save(self, commit: bool = True) -> User:
        user = super().save(commit=False)
        user.email = self.cleaned_data["email"]
        if commit:
            user.save()
        return user
```

[↑ Back to Contents](#contents)

---

## Protecting Views

### login_required and method_decorator

`login_required` is designed for function-based views. To use it with a class-based view built on `django.views.View` (with explicit `get` / `post` methods), wrap it with `method_decorator` applied to `dispatch`. Attaching to `dispatch` ensures it fires before Django routes the request to `get` or `post` — a single decorator covers all HTTP methods.

```python
from django.contrib.auth.decorators import login_required
from django.http import HttpRequest, HttpResponse
from django.utils.decorators import method_decorator
from django.views import View


@method_decorator(login_required, name="dispatch")           # redirects to settings.LOGIN_URL
class ClotheCreateView(View):
    def get(self, request: HttpRequest) -> HttpResponse:
        ...

    def post(self, request: HttpRequest) -> HttpResponse:
        ...


@method_decorator(
    login_required(login_url="/accounts/login/"),            # custom redirect
    name="dispatch",
)
class ClotheDeleteView(View):
    def get(self, request: HttpRequest, pk: int) -> HttpResponse:
        ...
```

### LoginRequiredMixin

Prefer `LoginRequiredMixin` over `method_decorator` when you want to configure `login_url` or `redirect_field_name` declaratively on the class. Works directly with `django.views.View`:

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.http import HttpRequest, HttpResponse
from django.views import View


class ClotheCreateView(LoginRequiredMixin, View):
    login_url = "/login/"
    redirect_field_name = "next"

    def get(self, request: HttpRequest) -> HttpResponse:
        ...

    def post(self, request: HttpRequest) -> HttpResponse:
        ...
```

Always put `LoginRequiredMixin` **first** in the MRO (before `View`).

### Checking Manually

Override `dispatch` on `View` to check authentication yourself:

```python
from django.http import HttpRequest, HttpResponse
from django.shortcuts import redirect
from django.views import View


class ClotheCreateView(View):
    def dispatch(self, request: HttpRequest, *args, **kwargs):
        if not request.user.is_authenticated:
            return redirect("login")
        return super().dispatch(request, *args, **kwargs)

    def get(self, request: HttpRequest) -> HttpResponse:
        ...

    def post(self, request: HttpRequest) -> HttpResponse:
        ...
```

[↑ Back to Contents](#contents)

---

## Permissions

### Built-in Permissions

Django auto-creates four permissions per model: `add_<model>`, `change_<model>`, `delete_<model>`, `view_<model>`.

```python
# In Python
request.user.has_perm("wardrobe.add_clothe")
request.user.has_perm("wardrobe.change_clothe")
request.user.has_perm("wardrobe.delete_clothe")

# In templates
{% if perms.wardrobe.add_clothe %}
  <a href="{% url 'wardrobe:create' %}">Add item</a>
{% endif %}
```

### permission_required and method_decorator

```python
from django.contrib.auth.decorators import permission_required
from django.http import HttpRequest, HttpResponse
from django.utils.decorators import method_decorator
from django.views import View


@method_decorator(
    permission_required("wardrobe.add_clothe", raise_exception=True),
    name="dispatch",
)
class ClotheCreateView(View):
    def get(self, request: HttpRequest) -> HttpResponse:
        ...

    def post(self, request: HttpRequest) -> HttpResponse:
        ...
```

`raise_exception=True` returns a 403 instead of redirecting to the login page.

### PermissionRequiredMixin

```python
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.http import HttpRequest, HttpResponse
from django.views import View


class ClotheCreateView(LoginRequiredMixin, PermissionRequiredMixin, View):
    permission_required = "wardrobe.add_clothe"
    raise_exception = True

    def get(self, request: HttpRequest) -> HttpResponse:
        ...

    def post(self, request: HttpRequest) -> HttpResponse:
        ...
```

### Custom Permissions

Define extra permissions in the model's `Meta`:

```python
class Clothe(models.Model):
    ...

    class Meta:
        permissions = [
            ("can_archive_clothe", "Can archive clothing items"),
            ("can_export_wardrobe", "Can export wardrobe to CSV"),
        ]
```

Use them exactly like built-in permissions:

```python
request.user.has_perm("wardrobe.can_archive_clothe")
```

[↑ Back to Contents](#contents)

---

## The request.user Object

```python
def my_view(request: HttpRequest) -> HttpResponse:
    user = request.user

    user.is_authenticated   # True for logged-in users, False for AnonymousUser
    user.is_anonymous       # opposite of is_authenticated
    user.is_active
    user.is_staff
    user.is_superuser

    user.username
    user.email
    user.get_full_name()    # "Ada Lovelace"
    user.get_short_name()   # "Ada"

    user.has_perm("app.permission_codename")
    user.has_perms(["app.perm1", "app.perm2"])
    user.has_module_perms("wardrobe")   # has any perm in the app
```

In templates, `request.user` is available automatically if `django.contrib.auth.context_processors.auth` is in `TEMPLATES[0]["OPTIONS"]["context_processors"]` (it is by default):

```html
{% if user.is_authenticated %} Hello, {{ user.username }}!
<a href="{% url 'logout' %}">Log out</a>
{% else %}
<a href="{% url 'login' %}">Log in</a>
{% endif %}
```

[↑ Back to Contents](#contents)

---

## Password Management

```python
from django.contrib.auth import update_session_auth_hash

# Change password programmatically
user.set_password("new_secure_password")
user.save()
update_session_auth_hash(request, user)  # keep the user logged in after password change
```

Using Django's built-in password change and reset views:

```python
from django.contrib.auth import views as auth_views

urlpatterns = [
    path("password/change/",          auth_views.PasswordChangeView.as_view(),        name="password_change"),
    path("password/change/done/",     auth_views.PasswordChangeDoneView.as_view(),    name="password_change_done"),
    path("password/reset/",           auth_views.PasswordResetView.as_view(),         name="password_reset"),
    path("password/reset/done/",      auth_views.PasswordResetDoneView.as_view(),     name="password_reset_done"),
    path("password/reset/<uidb64>/<token>/", auth_views.PasswordResetConfirmView.as_view(), name="password_reset_confirm"),
    path("password/reset/complete/",  auth_views.PasswordResetCompleteView.as_view(), name="password_reset_complete"),
]
```

These views require templates named after the view (e.g. `registration/password_reset_form.html`). Override `template_name` if you want a different path.

[↑ Back to Contents](#contents)

---

## Role-Based Users with Proxy Models

### The Problem This Solves

Imagine you're building a school platform. You have two kinds of users: **Teachers** and **Pupils**. Both need to log in. Both are users. But they have different roles, different permissions, and you want them to appear separately in the Django admin.

The naive solution is to add a `role` field to your User model and branch your logic on `if user.role == "teacher"`. This works but gets messy fast — your code becomes full of role checks, and the admin shows one giant undifferentiated user list.

A cleaner solution is **proxy models** combined with **Django Groups**.

---

### How Proxy Models Work

> **The core idea:** A proxy model is a Python class that pretends to be a different model, but shares the exact same database table as its parent.

When you create a `Teacher` and a `Pupil`, both rows land in the same `auth_user` table (or your custom user table). There is no separate `teacher` table or `pupil` table. The proxy model is just a Python-level alias with its own manager, methods, admin class, and group membership.

```
Database table: auth_user
┌────┬──────────┬───────────────────────────────┐
│ id │ username │ groups                        │
├────┼──────────┼───────────────────────────────┤
│  1 │ ms_smith │ ["Teachers"]                  │
│  2 │ tommy    │ ["Pupils"]                    │
│  3 │ mr_jones │ ["Teachers"]                  │
└────┴──────────┴───────────────────────────────┘

Python layer:
  Teacher.objects.all()  →  filters to group "Teachers"  →  [ms_smith, mr_jones]
  Pupil.objects.all()    →  filters to group "Pupils"    →  [tommy]
```

The distinction lives in the **Group** each user belongs to. The proxy model's manager applies that filter automatically.

---

### Step 1 — Define Roles with TextChoices

Keep role names in a `TextChoices` enum so there is **one source of truth** — models, signals, managers, forms, and templates all import from here instead of scattering magic strings everywhere.

```python
# accounts/models.py
from django.db import models


class Role(models.TextChoices):
    TEACHER = "Teachers", "Teachers"
    PUPIL   = "Pupils",   "Pupils"
```

`Role.values` → `["Teachers", "Pupils"]`  
`Role.TEACHER.value` → `"Teachers"`  
`Role.TEACHER.label` → `"Teachers"`

---

### Step 2 — Create the Groups via post_migrate Signal

Instead of a hand-edited migration file, use a `post_migrate` signal. The signal fires at the end of every `manage.py migrate` run and ensures the groups exist, regardless of whether it is the first migration or a later one.

```python
# accounts/apps.py
from django.apps import AppConfig
from django.db.models.signals import post_migrate


def create_user_groups(sender, **kwargs):
    """Ensure role groups exist after every migrate run."""
    from django.db.utils import OperationalError, ProgrammingError
    from django.contrib.auth.models import Group
    from .models import Role

    for role in Role.values:
        try:
            Group.objects.get_or_create(name=role)
        except (OperationalError, ProgrammingError):
            # The auth_group table doesn't exist yet (very first migrate run).
            pass


class AccountsConfig(AppConfig):
    name = "accounts"

    def ready(self):
        # Ensure groups exist after every migration.
        post_migrate.connect(create_user_groups, sender=self)

        # Wire up model signals (profile auto-creation, etc.).
        import accounts.signals  # noqa
```

> **On migration reversal:** `post_migrate` fires after both forward and reverse migrations. Because the handler uses `get_or_create` it is **purely additive** — it will not delete groups when you roll back. If you need groups removed on rollback, a data migration with `reverse_code` is the only option. For normal deployments, groups persisting after a reversal is harmless.

---

### Step 3 — Define the Proxy Models

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser, Group
from django.db import models


class Role(models.TextChoices):
    TEACHER = "Teachers", "Teachers"
    PUPIL   = "Pupils",   "Pupils"


class User(AbstractUser):
    """The real user model. All rows live here."""

    @property
    def is_teacher(self) -> bool:
        return self.groups.filter(name=Role.TEACHER.value).exists()

    @property
    def is_pupil(self) -> bool:
        return self.groups.filter(name=Role.PUPIL.value).exists()

    @property
    def profile(self):
        """Return the role-specific profile for this user, or None."""
        if self.is_teacher:
            return TeacherProfile.objects.filter(user_id=self.pk).first()
        if self.is_pupil:
            return PupilProfile.objects.filter(user_id=self.pk).first()
        return None


class Teacher(User):
    """A User who belongs to the 'Teachers' group."""

    class Meta:
        proxy = True
        verbose_name = "Teacher"
        verbose_name_plural = "Teachers"

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        self._assign_group(Role.TEACHER.value)

    def _assign_group(self, group_name: str) -> None:
        group, _ = Group.objects.get_or_create(name=group_name)
        self.groups.add(group)


class Pupil(User):
    """A User who belongs to the 'Pupils' group."""

    class Meta:
        proxy = True
        verbose_name = "Pupil"
        verbose_name_plural = "Pupils"

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        self._assign_group(Role.PUPIL.value)

    def _assign_group(self, group_name: str) -> None:
        group, _ = Group.objects.get_or_create(name=group_name)
        self.groups.add(group)
```

`proxy = True` is the only required Meta line. Everything else (methods, managers, admin) builds on top of it. `_assign_group` uses `get_or_create` — safe in tests or any environment where the `post_migrate` signal hasn't run yet.

---

### Step 4 — Proxy Model Managers

Right now `Teacher.objects.all()` returns every user, not just teachers. Fix this with a custom manager that pre-filters by group using `Role`:

```python
# accounts/models.py  (additions)

class TeacherManager(models.Manager):
    def get_queryset(self):
        return (
            super()
            .get_queryset()
            .filter(groups__name=Role.TEACHER.value)
            .distinct()
        )


class PupilManager(models.Manager):
    def get_queryset(self):
        return (
            super()
            .get_queryset()
            .filter(groups__name=Role.PUPIL.value)
            .distinct()
        )


class Teacher(User):
    objects = TeacherManager()

    class Meta:
        proxy = True
        verbose_name = "Teacher"
        verbose_name_plural = "Teachers"

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        self._assign_group(Role.TEACHER.value)

    def _assign_group(self, group_name: str) -> None:
        group, _ = Group.objects.get_or_create(name=group_name)
        self.groups.add(group)


class Pupil(User):
    objects = PupilManager()

    class Meta:
        proxy = True
        verbose_name = "Pupil"
        verbose_name_plural = "Pupils"

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        self._assign_group(Role.PUPIL.value)

    def _assign_group(self, group_name: str) -> None:
        group, _ = Group.objects.get_or_create(name=group_name)
        self.groups.add(group)
```

Now `Teacher.objects.all()` returns only users in the "Teachers" group, and `Pupil.objects.all()` returns only users in the "Pupils" group.

---

### Step 5 — Profile Models

Each proxy model gets a **OneToOne profile** for role-specific extra data. Because `Teacher` and `Pupil` are proxies of `User` (same DB table), the `OneToOneField` physically points at the `auth_user` table — Django handles this transparently.

```python
# accounts/models.py  (additions — add after the proxy model definitions)


class TeacherProfile(models.Model):
    user = models.OneToOneField(
        Teacher,
        on_delete=models.CASCADE,
        related_name="teacher_profile",
    )
    employee_number = models.CharField(max_length=20, blank=True)
    department      = models.CharField(max_length=100, blank=True)

    def __str__(self) -> str:
        return f"TeacherProfile — {self.user.get_full_name() or self.user.username}"


class PupilProfile(models.Model):
    user = models.OneToOneField(
        Pupil,
        on_delete=models.CASCADE,
        related_name="pupil_profile",
    )
    admission_number = models.CharField(max_length=20, blank=True)
    date_of_birth    = models.DateField(null=True, blank=True)

    def __str__(self) -> str:
        return f"PupilProfile — {self.user.get_full_name() or self.user.username}"
```

**Auto-create profiles on registration** using `post_save` signals. Put these in a dedicated `signals.py` — `AccountsConfig.ready()` (in `apps.py` above) already imports it:

```python
# accounts/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver

from .models import Teacher, Pupil, TeacherProfile, PupilProfile


@receiver(post_save, sender=Teacher)
def create_teacher_profile(sender, instance, created, **kwargs):
    if created:
        TeacherProfile.objects.get_or_create(user=instance)


@receiver(post_save, sender=Pupil)
def create_pupil_profile(sender, instance, created, **kwargs):
    if created:
        PupilProfile.objects.get_or_create(user=instance)
```

> **Why `sender=Teacher` works for registration:** When you save a `Teacher` instance (e.g. via `TeacherCreationForm.save()`), Django fires `post_save` with `sender=Teacher`. Updates that go through the session layer (password changes, toggling `is_active`) fire with `sender=User`, which is correct — those don't need to touch the profile.

**Accessing the profile from a view** — `request.user` is always a plain `User` instance, so the `teacher_profile` reverse accessor (which lives on the `Teacher` proxy class) is not directly reachable. Use the `profile` property added to `User` in Step 3, or query directly:

```python
from .models import TeacherProfile, PupilProfile


def some_view(request: HttpRequest) -> HttpResponse:
    # Via the convenience property on User (defined in Step 3):
    profile = request.user.profile   # returns TeacherProfile or PupilProfile or None

    # Or via direct query:
    profile = TeacherProfile.objects.get(user_id=request.user.pk)
```

In templates:

```html
{{ user.profile.department }}
{{ user.profile.date_of_birth }}
```

---

### Step 6 — Forms

Registration forms are the same as before — profile creation is now handled automatically by the `post_save` signal in Step 5 when `form.save()` is called. Add separate `ModelForm`s if you want an "edit profile" page.

```python
# accounts/forms.py
from django import forms
from django.contrib.auth.forms import UserCreationForm

from .models import Teacher, Pupil, TeacherProfile, PupilProfile


class TeacherCreationForm(UserCreationForm):
    """Registration form for teachers."""

    class Meta(UserCreationForm.Meta):
        model = Teacher
        fields = ["username", "first_name", "last_name", "email",
                  "password1", "password2"]

    def clean_email(self):
        email = self.cleaned_data.get("email")
        if email and Teacher.objects.filter(email=email).exists():
            raise forms.ValidationError("A teacher with this email already exists.")
        return email


class PupilCreationForm(UserCreationForm):
    """Registration form for pupils."""

    class Meta(UserCreationForm.Meta):
        model = Pupil
        fields = ["username", "first_name", "last_name", "email",
                  "password1", "password2"]

    def clean_email(self):
        email = self.cleaned_data.get("email")
        if email and Pupil.objects.filter(email=email).exists():
            raise forms.ValidationError("A pupil with this email already exists.")
        return email


class TeacherProfileForm(forms.ModelForm):
    """Edit form for a teacher's profile data."""

    class Meta:
        model = TeacherProfile
        fields = ["employee_number", "department"]


class PupilProfileForm(forms.ModelForm):
    """Edit form for a pupil's profile data."""

    class Meta:
        model = PupilProfile
        fields = ["admission_number", "date_of_birth"]
```

> **Why `clean_email` works:** `UserCreationForm` doesn't validate email uniqueness by default (username gets that check). Adding `clean_<fieldname>` is how Django forms validate a single field after its own validators — it can raise `ValidationError`.

---

### Step 7 — Views

Registration views use `django.views.View` with explicit `get` and `post` methods. Calling `form.save()` triggers the chain: `Teacher.save()` → group assigned → `post_save` → profile auto-created.

```python
# accounts/views.py
from django.contrib.auth import login
from django.http import HttpRequest, HttpResponse
from django.shortcuts import render, redirect
from django.views import View

from .forms import TeacherCreationForm, PupilCreationForm


class RegisterTeacherView(View):
    template_name = "accounts/register_teacher.html"

    def get(self, request: HttpRequest) -> HttpResponse:
        if request.user.is_authenticated:
            return redirect("home")
        return render(request, self.template_name, {"form": TeacherCreationForm()})

    def post(self, request: HttpRequest) -> HttpResponse:
        if request.user.is_authenticated:
            return redirect("home")
        form = TeacherCreationForm(request.POST)
        if form.is_valid():
            user = form.save()   # Teacher.save() → group → post_save → TeacherProfile
            login(request, user)
            return redirect("home")
        return render(request, self.template_name, {"form": form})


class RegisterPupilView(View):
    template_name = "accounts/register_pupil.html"

    def get(self, request: HttpRequest) -> HttpResponse:
        if request.user.is_authenticated:
            return redirect("home")
        return render(request, self.template_name, {"form": PupilCreationForm()})

    def post(self, request: HttpRequest) -> HttpResponse:
        if request.user.is_authenticated:
            return redirect("home")
        form = PupilCreationForm(request.POST)
        if form.is_valid():
            user = form.save()   # Pupil.save() → group → post_save → PupilProfile
            login(request, user)
            return redirect("home")
        return render(request, self.template_name, {"form": form})
```

```python
# accounts/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path("register/teacher/", views.RegisterTeacherView.as_view(), name="register_teacher"),
    path("register/pupil/",   views.RegisterPupilView.as_view(),   name="register_pupil"),
]
```

---

### Step 8 — Admin

Each proxy model gets its own admin section. Add a profile `StackedInline` so the profile fields appear directly on the teacher/pupil detail page.

```python
# accounts/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from django.contrib.auth.models import Group

from .models import User, Teacher, Pupil, TeacherProfile, PupilProfile


# ── Profile inlines ──────────────────────────────────────────────────────────

class TeacherProfileInline(admin.StackedInline):
    model = TeacherProfile
    can_delete = False
    verbose_name_plural = "Profile"


class PupilProfileInline(admin.StackedInline):
    model = PupilProfile
    can_delete = False
    verbose_name_plural = "Profile"


# ── Base User admin (optional — remove if you only want Teacher/Pupil) ───────

@admin.register(User)
class CustomUserAdmin(UserAdmin):
    """Catch-all for users not in Teacher or Pupil."""
    pass


# ── Teacher admin ─────────────────────────────────────────────────────────────

@admin.register(Teacher)
class TeacherAdmin(UserAdmin):
    """
    Admin for the Teacher proxy model.
    Only shows users in the 'Teachers' group.
    Profile fields appear inline.
    """
    inlines = [TeacherProfileInline]

    list_display  = ["username", "first_name", "last_name", "email", "is_active"]
    list_filter   = ["is_active"]
    search_fields = ["username", "first_name", "last_name", "email"]
    ordering      = ["last_name", "first_name"]

    fieldsets = [
        (None,            {"fields": ["username", "password"]}),
        ("Personal info", {"fields": ["first_name", "last_name", "email"]}),
        ("Status",        {"fields": ["is_active", "is_staff"], "classes": ["collapse"]}),
    ]

    add_fieldsets = [
        (
            None,
            {
                "classes": ["wide"],
                "fields": ["username", "first_name", "last_name", "email",
                           "password1", "password2"],
            },
        ),
    ]

    def get_queryset(self, request):
        return Teacher.objects.all()   # TeacherManager filters to group automatically

    def save_model(self, request, obj, form, change):
        super().save_model(request, obj, form, change)
        group, _ = Group.objects.get_or_create(name="Teachers")
        obj.groups.add(group)


# ── Pupil admin ───────────────────────────────────────────────────────────────

@admin.register(Pupil)
class PupilAdmin(UserAdmin):
    """
    Admin for the Pupil proxy model.
    Only shows users in the 'Pupils' group.
    Profile fields appear inline.
    """
    inlines = [PupilProfileInline]

    list_display  = ["username", "first_name", "last_name", "email", "is_active"]
    list_filter   = ["is_active"]
    search_fields = ["username", "first_name", "last_name", "email"]
    ordering      = ["last_name", "first_name"]

    fieldsets = [
        (None,            {"fields": ["username", "password"]}),
        ("Personal info", {"fields": ["first_name", "last_name", "email"]}),
        ("Status",        {"fields": ["is_active"], "classes": ["collapse"]}),
    ]

    add_fieldsets = [
        (
            None,
            {
                "classes": ["wide"],
                "fields": ["username", "first_name", "last_name", "email",
                           "password1", "password2"],
            },
        ),
    ]

    def get_queryset(self, request):
        return Pupil.objects.all()    # PupilManager filters to group automatically

    def save_model(self, request, obj, form, change):
        super().save_model(request, obj, form, change)
        group, _ = Group.objects.get_or_create(name="Pupils")
        obj.groups.add(group)
```

The admin sidebar shows three separate entries: **Users**, **Teachers**, and **Pupils**. Each teacher/pupil detail page includes its profile section inline.

---

### Step 9 — Checking Roles in Code

**Check by group membership (most explicit):**

```python
from .models import Role

def my_view(request: HttpRequest) -> HttpResponse:
    user = request.user

    if user.groups.filter(name=Role.TEACHER.value).exists():
        ...   # user is a teacher

    if user.groups.filter(name=Role.PUPIL.value).exists():
        ...   # user is a pupil
```

**Use the helper properties on User (cleaner — defined in Step 3):**

```python
if request.user.is_teacher:
    ...
```

```html
<!-- In a template -->
{% if user.is_teacher %}
  <a href="{% url 'teacher_dashboard' %}">My Classes</a>
{% endif %}
```

**Protect a view using `UserPassesTestMixin` with `View`:**

```python
from django.contrib.auth.mixins import LoginRequiredMixin, UserPassesTestMixin
from django.http import HttpRequest, HttpResponse
from django.shortcuts import render
from django.views import View

from .models import Pupil, Role


class PupilListView(LoginRequiredMixin, UserPassesTestMixin, View):
    """Only teachers can see the list of pupils."""

    def test_func(self) -> bool:
        return self.request.user.groups.filter(name=Role.TEACHER.value).exists()

    def get(self, request: HttpRequest) -> HttpResponse:
        pupils = Pupil.objects.all()
        return render(request, "accounts/pupil_list.html", {"pupils": pupils})
```

**Protect a view using `method_decorator` + `user_passes_test`:**

```python
from django.contrib.auth.decorators import login_required, user_passes_test
from django.http import HttpRequest, HttpResponse
from django.shortcuts import render
from django.utils.decorators import method_decorator
from django.views import View

from .models import Teacher, Role


def is_teacher(user) -> bool:
    return user.groups.filter(name=Role.TEACHER.value).exists()


@method_decorator(
    [login_required, user_passes_test(is_teacher, login_url="/forbidden/")],
    name="dispatch",
)
class TeacherDashboardView(View):
    def get(self, request: HttpRequest) -> HttpResponse:
        teachers = Teacher.objects.all()
        return render(request, "accounts/teacher_dashboard.html", {"teachers": teachers})
```

> `method_decorator` accepts a list of decorators — they are applied in order from the outside in, matching how stacked `@decorator` lines work on a plain function.

---

### What to Watch Out For

**`request.user` is always a plain `User`, never a `Teacher` or `Pupil`.**
Django's auth system loads the user from the session as the base `User` model. So `isinstance(request.user, Teacher)` will be `False` even if that user is in the Teachers group. Use group checks (`user.groups.filter(...)`) or the `is_teacher` / `is_pupil` properties instead.

**Profile access from `request.user` requires a query.**
The `teacher_profile` reverse accessor lives on the `Teacher` proxy class, not on `User`. Since `request.user` is a `User` instance, call `request.user.profile` (the property added in Step 3) or query directly with `TeacherProfile.objects.get(user_id=request.user.pk)`. Never do `request.user.teacher_profile` — it will raise `AttributeError`.

**`post_migrate` signal vs data migration — what happens on reversal.**
The `post_migrate` handler uses `get_or_create` and is additive only. Rolling back a migration still fires `post_migrate`, and the handler will re-create any groups it finds missing — it will **never delete** groups. If you need groups cleaned up on rollback (rare in practice), you must use a data migration with `reverse_code` instead.

**The manager filter uses a JOIN.**
`Teacher.objects.all()` produces a query with a JOIN on `auth_user_groups`. If you chain further filters, Django handles this correctly, but be aware that joins through M2M can produce duplicate rows in some edge cases. The `.distinct()` call in the manager prevents this.

**Migrations for proxy models are minimal.**
Running `makemigrations` after adding a proxy model creates a migration that registers it with Django's content type framework (needed for permissions and admin), but creates no new database table. This is expected and correct.

**Don't assign a user to both groups.**
The proxy model pattern assumes a user belongs to exactly one role group. If your app needs multi-role users, prefer a different approach (a `role` field or a profile model with a FK to User).

**Always use `get_or_create` when assigning groups in `save()`.**
Using `Group.objects.get()` raises `DoesNotExist` if the signal hasn't run yet (e.g. in tests). `get_or_create` is always safe.

[↑ Back to Contents](#contents)

---

## References

- https://docs.djangoproject.com/en/stable/topics/auth/
- https://docs.djangoproject.com/en/stable/topics/auth/default/
- https://docs.djangoproject.com/en/stable/topics/auth/customizing/
- https://docs.djangoproject.com/en/stable/ref/contrib/auth/
- https://docs.djangoproject.com/en/stable/topics/db/models/#proxy-models
- https://docs.djangoproject.com/en/stable/topics/auth/default/#groups
- https://docs.djangoproject.com/en/stable/ref/signals/#post-migrate
