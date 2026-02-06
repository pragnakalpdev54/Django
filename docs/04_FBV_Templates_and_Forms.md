# Part 4: Function-Based Views - Templates & Forms

## Django Template System (HTML-Based)

**Important Note:** Django uses its own template language (DTL - Django Template Language), NOT Jinja2. While they look similar, Django templates are HTML-based with special template tags.

Django's template system allows you to create dynamic HTML pages by combining static HTML with dynamic data from views.

### Template Syntax (Django Template Language)

**Variables** - Display dynamic data
```django
{{ variable }}
{{ user.username }}
{{ task.title }}
```

**Tags** - Control logic and flow
```django
{% tag %}
{% for item in items %}
{% if condition %}
{% url 'view_name' %}
```

**Filters** - Transform data
```django
{{ value|filter }}
{{ text|lower }}
{{ date|date:"Y-m-d" }}
{{ number|add:5 }}
```

**Comments**
```django
{# Single line comment #}
{% comment %}
Multi-line comment
{% endcomment %}
```

**Key Difference from Jinja2:**
- Django: `{% url 'view_name' %}` 
- Jinja2: `{{ url_for('view_name') }}`
- Django templates are HTML files with Django template tags embedded

---

## Creating Templates

### Directory Structure

Create the following structure:

```
tasks/
└── templates/
    └── tasks/
        ├── base.html
        ├── task_list.html
        ├── task_detail.html
        ├── task_form.html
        ├── task_confirm_delete.html
        ├── login.html
        └── register.html
```

### Base Template (Template Inheritance)

**File: `tasks/templates/tasks/base.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Task Manager{% endblock %}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.0/font/bootstrap-icons.css">
    <style>
        :root {
            --primary-color: #4f46e5;
            --secondary-color: #7c3aed;
        }
        body {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        .navbar {
            background: rgba(255, 255, 255, 0.95) !important;
            backdrop-filter: blur(10px);
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        .main-content {
            margin-top: 80px;
            padding-bottom: 50px;
        }
        .card {
            border: none;
            border-radius: 15px;
            box-shadow: 0 5px 20px rgba(0,0,0,0.1);
            transition: transform 0.3s ease;
        }
        .card:hover {
            transform: translateY(-5px);
        }
        .badge-priority-high {
            background-color: #dc3545;
        }
        .badge-priority-medium {
            background-color: #ffc107;
            color: #000;
        }
        .badge-priority-low {
            background-color: #28a745;
        }
        .task-completed {
            opacity: 0.7;
        }
    </style>
    {% block extra_css %}{% endblock %}
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-light fixed-top">
        <div class="container">
            <a class="navbar-brand fw-bold" href="{% url 'task_list' %}">
                <i class="bi bi-check2-square"></i> Task Manager
            </a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav ms-auto">
                    {% if user.is_authenticated %}
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'task_list' %}">
                                <i class="bi bi-list-task"></i> My Tasks
                            </a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'task_create' %}">
                                <i class="bi bi-plus-circle"></i> New Task
                            </a>
                        </li>
                        <li class="nav-item">
                            <span class="nav-link">
                                <i class="bi bi-person-circle"></i> {{ user.username }}
                            </span>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'logout' %}">
                                <i class="bi bi-box-arrow-right"></i> Logout
                            </a>
                        </li>
                    {% else %}
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'login' %}">Login</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" href="{% url 'register' %}">Register</a>
                        </li>
                    {% endif %}
                </ul>
            </div>
        </div>
    </nav>

    <div class="container main-content">
        {% if messages %}
            {% for message in messages %}
                <div class="alert alert-{{ message.tags }} alert-dismissible fade show" role="alert">
                    {{ message }}
                    <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
                </div>
            {% endfor %}
        {% endif %}

        {% block content %}
        {% endblock %}
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    {% block extra_js %}{% endblock %}
</body>
</html>
```

**Key Concepts:**

1. **Block Tags**: `{% block title %}` - Define sections that child templates can override
2. **URL Tag**: `{% url 'task_list' %}` - Generate URLs by name
3. **If Statement**: `{% if user.is_authenticated %}` - Conditional rendering
4. **For Loop**: `{% for message in messages %}` - Iterate over items
5. **Variable**: `{{ user.username }}` - Display variable value

---

### Task List Template

**File: `tasks/templates/tasks/task_list.html`**

```html
{% extends 'tasks/base.html' %}

{% block title %}My Tasks - Task Manager{% endblock %}

{% block content %}
<div class="row mb-4">
    <div class="col-md-12">
        <div class="card">
            <div class="card-body">
                <h2 class="card-title mb-4">
                    <i class="bi bi-list-check"></i> My Tasks
                </h2>
                
                <!-- Search and Filter Form -->
                <form method="GET" class="row g-3 mb-4">
                    <div class="col-md-4">
                        <input type="text" name="search" class="form-control" 
                               placeholder="Search tasks..." value="{{ search_query|default:'' }}">
                    </div>
                    <div class="col-md-3">
                        <select name="status" class="form-control">
                            <option value="">All Status</option>
                            <option value="pending" {% if status_filter == 'pending' %}selected{% endif %}>
                                Pending
                            </option>
                            <option value="in_progress" {% if status_filter == 'in_progress' %}selected{% endif %}>
                                In Progress
                            </option>
                            <option value="completed" {% if status_filter == 'completed' %}selected{% endif %}>
                                Completed
                            </option>
                        </select>
                    </div>
                    <div class="col-md-3">
                        <select name="priority" class="form-control">
                            <option value="">All Priorities</option>
                            <option value="high" {% if priority_filter == 'high' %}selected{% endif %}>High</option>
                            <option value="medium" {% if priority_filter == 'medium' %}selected{% endif %}>Medium</option>
                            <option value="low" {% if priority_filter == 'low' %}selected{% endif %}>Low</option>
                        </select>
                    </div>
                    <div class="col-md-2">
                        <button type="submit" class="btn btn-primary w-100">
                            <i class="bi bi-search"></i> Filter
                        </button>
                    </div>
                </form>

                <!-- Task Statistics -->
                <div class="row mb-4">
                    <div class="col-md-4">
                        <div class="card bg-primary text-white">
                            <div class="card-body text-center">
                                <h5>Total Tasks</h5>
                                <h2>{{ tasks.count }}</h2>
                            </div>
                        </div>
                    </div>
                    <div class="col-md-4">
                        <div class="card bg-success text-white">
                            <div class="card-body text-center">
                                <h5>Completed</h5>
                                <h2>{{ tasks|length }}</h2>
                            </div>
                        </div>
                    </div>
                    <div class="col-md-4">
                        <div class="card bg-warning text-white">
                            <div class="card-body text-center">
                                <h5>Pending</h5>
                                <h2>{{ tasks|length }}</h2>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Task List -->
                {% if tasks %}
                    <div class="row">
                        {% for task in tasks %}
                            <div class="col-md-6 mb-3">
                                <div class="card {% if task.is_completed %}task-completed border-success{% endif %}">
                                    <div class="card-body">
                                        <div class="d-flex justify-content-between align-items-start mb-2">
                                            <h5 class="card-title {% if task.is_completed %}text-decoration-line-through text-muted{% endif %}">
                                                {{ task.title }}
                                            </h5>
                                            <span class="badge badge-priority-{{ task.priority }}">
                                                {{ task.get_priority_display }}
                                            </span>
                                        </div>
                                        
                                        <p class="card-text text-muted">
                                            {{ task.description|truncatewords:20 }}
                                        </p>
                                        
                                        <div class="mb-2">
                                            <span class="badge bg-info">{{ task.get_status_display }}</span>
                                            {% if task.due_date %}
                                                <span class="badge bg-secondary">
                                                    <i class="bi bi-calendar"></i> {{ task.due_date|date:"M d, Y" }}
                                                </span>
                                            {% endif %}
                                        </div>
                                        
                                        <div class="btn-group btn-group-sm" role="group">
                                            <a href="{% url 'task_detail' task.pk %}" class="btn btn-info">
                                                <i class="bi bi-eye"></i> View
                                            </a>
                                            <a href="{% url 'task_update' task.pk %}" class="btn btn-warning">
                                                <i class="bi bi-pencil"></i> Edit
                                            </a>
                                            <a href="{% url 'task_toggle_complete' task.pk %}" class="btn btn-success">
                                                <i class="bi bi-check-circle"></i> 
                                                {% if task.is_completed %}Undo{% else %}Done{% endif %}
                                            </a>
                                            <a href="{% url 'task_delete' task.pk %}" class="btn btn-danger">
                                                <i class="bi bi-trash"></i>
                                            </a>
                                        </div>
                                    </div>
                                </div>
                            </div>
                        {% endfor %}
                    </div>
                {% else %}
                    <div class="alert alert-info text-center">
                        <i class="bi bi-info-circle"></i> No tasks found. 
                        <a href="{% url 'task_create' %}" class="alert-link">Create your first task!</a>
                    </div>
                {% endif %}
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

**Template Features Explained:**

1. **Template Inheritance**: `{% extends 'tasks/base.html' %}`
2. **Filters**: `{{ task.description|truncatewords:20 }}` - Limit words
3. **Date Formatting**: `{{ task.due_date|date:"M d, Y" }}`
4. **Default Values**: `{{ search_query|default:'' }}`
5. **Method Calls**: `{{ task.get_priority_display }}`
6. **Conditional Classes**: `{% if task.is_completed %}task-completed{% endif %}`

---

### Task Detail Template

**File: `tasks/templates/tasks/task_detail.html`**

```html
{% extends 'tasks/base.html' %}

{% block title %}{{ task.title }} - Task Manager{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-8">
        <div class="card">
            <div class="card-body">
                <div class="d-flex justify-content-between align-items-start mb-3">
                    <h2 class="card-title">{{ task.title }}</h2>
                    <span class="badge badge-priority-{{ task.priority }} fs-6">
                        {{ task.get_priority_display }}
                    </span>
                </div>
                
                <div class="mb-3">
                    <span class="badge bg-info fs-6">{{ task.get_status_display }}</span>
                    {% if task.is_completed %}
                        <span class="badge bg-success fs-6">
                            <i class="bi bi-check-circle"></i> Completed
                        </span>
                    {% endif %}
                </div>
                
                <div class="mb-4">
                    <h5>Description:</h5>
                    <p class="text-muted">
                        {% if task.description %}
                            {{ task.description|linebreaks }}
                        {% else %}
                            <em>No description provided</em>
                        {% endif %}
                    </p>
                </div>
                
                <div class="row mb-4">
                    <div class="col-md-6">
                        <p><strong>Created By:</strong> {{ task.created_by.username }}</p>
                        <p><strong>Created At:</strong> {{ task.created_at|date:"F d, Y H:i" }}</p>
                    </div>
                    <div class="col-md-6">
                        <p><strong>Updated At:</strong> {{ task.updated_at|date:"F d, Y H:i" }}</p>
                        {% if task.due_date %}
                            <p><strong>Due Date:</strong> {{ task.due_date|date:"F d, Y" }}</p>
                        {% endif %}
                    </div>
                </div>
                
                <div class="btn-group" role="group">
                    <a href="{% url 'task_update' task.pk %}" class="btn btn-warning">
                        <i class="bi bi-pencil"></i> Edit
                    </a>
                    <a href="{% url 'task_toggle_complete' task.pk %}" class="btn btn-success">
                        <i class="bi bi-check-circle"></i> 
                        {% if task.is_completed %}Mark Incomplete{% else %}Mark Complete{% endif %}
                    </a>
                    <a href="{% url 'task_delete' task.pk %}" class="btn btn-danger">
                        <i class="bi bi-trash"></i> Delete
                    </a>
                    <a href="{% url 'task_list' %}" class="btn btn-secondary">
                        <i class="bi bi-arrow-left"></i> Back
                    </a>
                </div>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

**New Filter:**
- `linebreaks`: Converts newlines to `<br>` and `<p>` tags

---

## Django Forms

Forms handle user input, validation, and data cleaning.

**Note:** We already created the `TaskForm` in the previous section ([`03_FBV_Views_and_URLs.md`](03_FBV_Views_and_URLs.md)). In this section, we'll expand on forms and create additional forms for user authentication.

### Expanding the forms.py File

Open the existing `tasks/forms.py` file and add the user registration form:

```python
from django import forms
from django.contrib.auth.models import User
from django.contrib.auth.forms import UserCreationForm
from .models import Task

# TaskForm already created in previous section
class TaskForm(forms.ModelForm):
    """Form for creating and updating tasks"""
    
    class Meta:
        model = Task
        fields = ['title', 'description', 'priority', 'status', 'due_date']
        widgets = {
            'title': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'Enter task title',
                'required': True
            }),
            'description': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 4,
                'placeholder': 'Enter task description'
            }),
            'priority': forms.Select(attrs={
                'class': 'form-control'
            }),
            'status': forms.Select(attrs={
                'class': 'form-control'
            }),
            'due_date': forms.DateInput(attrs={
                'class': 'form-control',
                'type': 'date'
            }),
        }
        labels = {
            'title': 'Task Title',
            'description': 'Description',
            'priority': 'Priority Level',
            'status': 'Current Status',
            'due_date': 'Due Date',
        }
        help_texts = {
            'title': 'Enter a clear, concise title for your task',
            'due_date': 'Optional: Set a deadline for this task',
        }
    
    def clean_title(self):
        title = self.cleaned_data.get('title')
        if len(title) < 3:
            raise forms.ValidationError("Title must be at least 3 characters long")
        return title
    
    def clean_due_date(self):
        due_date = self.cleaned_data.get('due_date')
        if due_date:
            from django.utils import timezone
            if due_date < timezone.now().date():
                raise forms.ValidationError("Due date cannot be in the past")
        return due_date

# Add User Registration Form below

class UserRegisterForm(UserCreationForm):
    email = forms.EmailField(
        required=True,
        widget=forms.EmailInput(attrs={
            'class': 'form-control',
            'placeholder': 'Email address'
        })
    )
    
    class Meta:
        model = User
        fields = ['username', 'email', 'password1', 'password2']
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fields['username'].widget.attrs.update({
            'class': 'form-control',
            'placeholder': 'Username'
        })
        self.fields['password1'].widget.attrs.update({
            'class': 'form-control',
            'placeholder': 'Password'
        })
        self.fields['password2'].widget.attrs.update({
            'class': 'form-control',
            'placeholder': 'Confirm Password'
        })
    
    def clean_email(self):
        email = self.cleaned_data.get('email')
        if User.objects.filter(email=email).exists():
            raise forms.ValidationError("This email is already registered")
        return email
```

### Form Types

**ModelForm**
- Automatically creates form from model
- Handles model saving
- Best for CRUD operations

**Form**
- Manual field definition
- More control
- Use for non-model forms (search, contact, etc.)

### Form Validation

**Field-Level Validation**
```python
def clean_fieldname(self):
    data = self.cleaned_data.get('fieldname')
    # Validate data
    if not valid:
        raise forms.ValidationError("Error message")
    return data
```

**Form-Level Validation**
```python
def clean(self):
    cleaned_data = super().clean()
    field1 = cleaned_data.get('field1')
    field2 = cleaned_data.get('field2')
    
    if field1 and field2:
        # Cross-field validation
        if field1 > field2:
            raise forms.ValidationError("Field1 must be less than Field2")
    
    return cleaned_data
```

---

### Task Form Template

**File: `tasks/templates/tasks/task_form.html`**

```html
{% extends 'tasks/base.html' %}

{% block title %}{{ action }} Task - Task Manager{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-8">
        <div class="card">
            <div class="card-body">
                <h2 class="card-title mb-4">
                    <i class="bi bi-{% if action == 'Create' %}plus-circle{% else %}pencil{% endif %}"></i> 
                    {{ action }} Task
                </h2>
                
                <form method="POST" novalidate>
                    {% csrf_token %}
                    
                    {% if form.non_field_errors %}
                        <div class="alert alert-danger">
                            {{ form.non_field_errors }}
                        </div>
                    {% endif %}
                    
                    {% for field in form %}
                        <div class="mb-3">
                            <label for="{{ field.id_for_label }}" class="form-label">
                                {{ field.label }}
                                {% if field.field.required %}
                                    <span class="text-danger">*</span>
                                {% endif %}
                            </label>
                            {{ field }}
                            {% if field.errors %}
                                <div class="text-danger small mt-1">
                                    {% for error in field.errors %}
                                        <div>{{ error }}</div>
                                    {% endfor %}
                                </div>
                            {% endif %}
                            {% if field.help_text %}
                                <small class="form-text text-muted d-block">{{ field.help_text }}</small>
                            {% endif %}
                        </div>
                    {% endfor %}
                    
                    <div class="d-flex gap-2">
                        <button type="submit" class="btn btn-primary">
                            <i class="bi bi-save"></i> {{ action }} Task
                        </button>
                        <a href="{% url 'task_list' %}" class="btn btn-secondary">
                            <i class="bi bi-x-circle"></i> Cancel
                        </a>
                    </div>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

**Form Template Features:**

1. **CSRF Token**: `{% csrf_token %}` - Security against CSRF attacks (required!)
2. **Form Iteration**: `{% for field in form %}` - Loop through all fields
3. **Field Properties**: `{{ field.label }}`, `{{ field.errors }}`, `{{ field.help_text }}`
4. **Error Display**: Show validation errors
5. **Required Fields**: Mark with asterisk

---

### Delete Confirmation Template

**File: `tasks/templates/tasks/task_confirm_delete.html`**

```html
{% extends 'tasks/base.html' %}

{% block title %}Delete Task - Task Manager{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-6">
        <div class="card border-danger">
            <div class="card-body">
                <h2 class="card-title text-danger mb-4">
                    <i class="bi bi-exclamation-triangle"></i> Confirm Delete
                </h2>
                
                <p class="lead">Are you sure you want to delete this task?</p>
                
                <div class="alert alert-warning">
                    <strong>Task:</strong> {{ task.title }}<br>
                    <strong>Priority:</strong> {{ task.get_priority_display }}<br>
                    <strong>Status:</strong> {{ task.get_status_display }}
                </div>
                
                <p class="text-danger">
                    <i class="bi bi-info-circle"></i> This action cannot be undone.
                </p>
                
                <form method="POST">
                    {% csrf_token %}
                    <div class="d-flex gap-2">
                        <button type="submit" class="btn btn-danger">
                            <i class="bi bi-trash"></i> Yes, Delete
                        </button>
                        <a href="{% url 'task_list' %}" class="btn btn-secondary">
                            <i class="bi bi-x-circle"></i> Cancel
                        </a>
                    </div>
                </form>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

---

### Authentication Templates

**File: `tasks/templates/tasks/login.html`**

```html
{% extends 'tasks/base.html' %}

{% block title %}Login - Task Manager{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-5">
        <div class="card">
            <div class="card-body">
                <h2 class="card-title text-center mb-4">
                    <i class="bi bi-box-arrow-in-right"></i> Login
                </h2>
                
                <form method="POST">
                    {% csrf_token %}
                    <div class="mb-3">
                        <label for="username" class="form-label">Username</label>
                        <input type="text" name="username" id="username" class="form-control" required>
                    </div>
                    <div class="mb-3">
                        <label for="password" class="form-label">Password</label>
                        <input type="password" name="password" id="password" class="form-control" required>
                    </div>
                    <button type="submit" class="btn btn-primary w-100">
                        <i class="bi bi-box-arrow-in-right"></i> Login
                    </button>
                </form>
                
                <hr>
                <p class="text-center mb-0">
                    Don't have an account? 
                    <a href="{% url 'register' %}">Register here</a>
                </p>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

**File: `tasks/templates/tasks/register.html`**

```html
{% extends 'tasks/base.html' %}

{% block title %}Register - Task Manager{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-6">
        <div class="card">
            <div class="card-body">
                <h2 class="card-title text-center mb-4">
                    <i class="bi bi-person-plus"></i> Create Account
                </h2>
                
                <form method="POST" novalidate>
                    {% csrf_token %}
                    
                    {% if form.non_field_errors %}
                        <div class="alert alert-danger">
                            {{ form.non_field_errors }}
                        </div>
                    {% endif %}
                    
                    {% for field in form %}
                        <div class="mb-3">
                            <label for="{{ field.id_for_label }}" class="form-label">
                                {{ field.label }}
                            </label>
                            {{ field }}
                            {% if field.errors %}
                                <div class="text-danger small mt-1">
                                    {% for error in field.errors %}
                                        <div>{{ error }}</div>
                                    {% endfor %}
                                </div>
                            {% endif %}
                            {% if field.help_text %}
                                <small class="form-text text-muted d-block">{{ field.help_text }}</small>
                            {% endif %}
                        </div>
                    {% endfor %}
                    
                    <button type="submit" class="btn btn-primary w-100">
                        <i class="bi bi-person-plus"></i> Register
                    </button>
                </form>
                
                <hr>
                <p class="text-center mb-0">
                    Already have an account? 
                    <a href="{% url 'login' %}">Login here</a>
                </p>
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

---

## Common Template Filters

### String Filters

```django
{{ value|lower }}              {# Convert to lowercase #}
{{ value|upper }}              {# Convert to uppercase #}
{{ value|title }}              {# Title Case #}
{{ value|capfirst }}           {# Capitalize first letter #}
{{ value|truncatewords:10 }}   {# Limit to 10 words #}
{{ value|truncatechars:50 }}   {# Limit to 50 characters #}
{{ value|linebreaks }}         {# Convert newlines to <p> and <br> #}
{{ value|striptags }}          {# Remove HTML tags #}
{{ value|slugify }}            {# Convert to slug #}
```

### Number Filters

```django
{{ value|add:5 }}              {# Add 5 #}
{{ value|floatformat:2 }}      {# Format to 2 decimal places #}
```

### Date Filters

```django
{{ value|date:"Y-m-d" }}       {# 2024-01-23 #}
{{ value|date:"F d, Y" }}      {# January 23, 2024 #}
{{ value|time:"H:i" }}         {# 14:30 #}
{{ value|timesince }}          {# "2 hours ago" #}
{{ value|timeuntil }}          {# "in 3 days" #}
```

### List Filters

```django
{{ value|length }}             {# Number of items #}
{{ value|first }}              {# First item #}
{{ value|last }}               {# Last item #}
{{ value|join:", " }}          {# Join with comma #}
{{ value|slice:":5" }}         {# First 5 items #}
```

### Default Filters

```django
{{ value|default:"N/A" }}      {# Show "N/A" if empty #}
{{ value|default_if_none:"N/A" }} {# Show "N/A" if None #}
```

---

## Template Tags

### Control Flow

```django
{% if condition %}
    ...
{% elif other_condition %}
    ...
{% else %}
    ...
{% endif %}

{% for item in items %}
    {{ item }}
{% empty %}
    No items found
{% endfor %}
```

### Loop Variables

```django
{% for task in tasks %}
    {{ forloop.counter }}      {# 1, 2, 3, ... #}
    {{ forloop.counter0 }}     {# 0, 1, 2, ... #}
    {{ forloop.first }}        {# True on first iteration #}
    {{ forloop.last }}         {# True on last iteration #}
    {{ forloop.parentloop }}   {# Access parent loop #}
{% endfor %}
```

### Include

```django
{% include 'tasks/task_card.html' with task=task %}
```

### Static Files

```django
{% load static %}
<link rel="stylesheet" href="{% static 'css/style.css' %}">
<img src="{% static 'images/logo.png' %}">
```

---

## AJAX for API Requests

### Why AJAX?

AJAX (Asynchronous JavaScript and XML) allows you to:
- Update parts of a page without full reload
- Make API calls to Django views
- Improve user experience
- Handle PATCH and DELETE methods from the browser

### AJAX Setup in Base Template

Add this to your base template before closing `</body>`:

```html
<script>
// CSRF Token Setup for AJAX
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');

// Default AJAX settings
const ajaxDefaults = {
    headers: {
        'X-CSRFToken': csrftoken,
        'Accept': 'application/json',
        'Content-Type': 'application/json'
    }
};
</script>
```

### Example 1: Update Task with AJAX (PATCH)

**HTML Template:**
```html
<button onclick="updateTask({{ task.id }})" class="btn btn-warning">
    <i class="bi bi-pencil"></i> Quick Update
</button>

<script>
async function updateTask(taskId) {
    const title = prompt('Enter new title:');
    if (!title) return;
    
    try {
        const response = await fetch(`/task/${taskId}/update/`, {
            method: 'PATCH',
            headers: {
                'X-CSRFToken': csrftoken,
                'Accept': 'application/json',
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                title: title
            })
        });
        
        const data = await response.json();
        
        if (response.ok) {
            alert(data.message);
            location.reload(); // Refresh to see changes
        } else {
            alert('Error: ' + JSON.stringify(data.error));
        }
    } catch (error) {
        console.error('Error:', error);
        alert('Failed to update task');
    }
}
</script>
```

### Example 2: Delete Task with AJAX (DELETE)

**HTML Template:**
```html
<button onclick="deleteTask({{ task.id }})" class="btn btn-danger">
    <i class="bi bi-trash"></i> Delete
</button>

<script>
async function deleteTask(taskId) {
    if (!confirm('Are you sure you want to delete this task?')) {
        return;
    }
    
    try {
        const response = await fetch(`/task/${taskId}/delete/`, {
            method: 'DELETE',
            headers: {
                'X-CSRFToken': csrftoken,
                'Accept': 'application/json'
            }
        });
        
        const data = await response.json();
        
        if (response.ok) {
            alert(data.message);
            // Remove task card from DOM
            document.getElementById(`task-${taskId}`).remove();
        } else {
            alert('Error: ' + data.error);
        }
    } catch (error) {
        console.error('Error:', error);
        alert('Failed to delete task');
    }
}
</script>
```

### Example 3: Toggle Task Completion with AJAX

**HTML Template:**
```html
<button onclick="toggleComplete({{ task.id }}, {{ task.is_completed|lower }})" 
        class="btn btn-success" id="toggle-btn-{{ task.id }}">
    <i class="bi bi-check-circle"></i> 
    <span id="toggle-text-{{ task.id }}">
        {% if task.is_completed %}Undo{% else %}Done{% endif %}
    </span>
</button>

<script>
async function toggleComplete(taskId, isCompleted) {
    try {
        const response = await fetch(`/task/${taskId}/toggle/`, {
            method: 'POST',
            headers: {
                'X-CSRFToken': csrftoken,
                'Accept': 'application/json'
            }
        });
        
        const data = await response.json();
        
        if (response.ok) {
            // Update button text
            const textElement = document.getElementById(`toggle-text-${taskId}`);
            textElement.textContent = isCompleted ? 'Done' : 'Undo';
            
            // Update task card styling
            const taskCard = document.getElementById(`task-${taskId}`);
            taskCard.classList.toggle('task-completed');
            
            // Show success message
            showToast(data.message, 'success');
        } else {
            alert('Error: ' + data.error);
        }
    } catch (error) {
        console.error('Error:', error);
        alert('Failed to toggle task');
    }
}
</script>
```

### Example 4: Live Search with AJAX

**HTML Template:**
```html
<input type="text" id="search-input" class="form-control" 
       placeholder="Search tasks..." onkeyup="liveSearch()">

<div id="search-results"></div>

<script>
let searchTimeout;

function liveSearch() {
    clearTimeout(searchTimeout);
    
    searchTimeout = setTimeout(async () => {
        const query = document.getElementById('search-input').value;
        
        if (query.length < 2) {
            document.getElementById('search-results').innerHTML = '';
            return;
        }
        
        try {
            const response = await fetch(`/tasks/?search=all&search_by=${encodeURIComponent(query)}`, {
                headers: {
                    'Accept': 'application/json'
                }
            });
            
            const data = await response.json();
            
            if (response.ok) {
                displaySearchResults(data.tasks);
            }
        } catch (error) {
            console.error('Error:', error);
        }
    }, 300); // Wait 300ms after user stops typing
}

function displaySearchResults(tasks) {
    const resultsDiv = document.getElementById('search-results');
    
    if (tasks.length === 0) {
        resultsDiv.innerHTML = '<p class="text-muted">No tasks found</p>';
        return;
    }
    
    let html = '<div class="list-group">';
    tasks.forEach(task => {
        html += `
            <a href="/task/${task.id}/" class="list-group-item list-group-item-action">
                <h6>${task.title}</h6>
                <small class="text-muted">${task.description}</small>
            </a>
        `;
    });
    html += '</div>';
    
    resultsDiv.innerHTML = html;
}
</script>
```

### Example 5: Create Task with AJAX

**HTML Template:**
```html
<form id="task-form" onsubmit="createTask(event)">
    <div class="mb-3">
        <input type="text" id="task-title" class="form-control" 
               placeholder="Task title" required>
    </div>
    <div class="mb-3">
        <textarea id="task-description" class="form-control" 
                  placeholder="Description"></textarea>
    </div>
    <div class="mb-3">
        <select id="task-priority" class="form-control">
            <option value="low">Low</option>
            <option value="medium" selected>Medium</option>
            <option value="high">High</option>
        </select>
    </div>
    <button type="submit" class="btn btn-primary">Create Task</button>
</form>

<script>
async function createTask(event) {
    event.preventDefault();
    
    const formData = {
        title: document.getElementById('task-title').value,
        description: document.getElementById('task-description').value,
        priority: document.getElementById('task-priority').value,
        status: 'pending'
    };
    
    try {
        const response = await fetch('/task/create/', {
            method: 'POST',
            headers: {
                'X-CSRFToken': csrftoken,
                'Accept': 'application/json',
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(formData)
        });
        
        const data = await response.json();
        
        if (response.ok) {
            showToast('Task created successfully!', 'success');
            document.getElementById('task-form').reset();
            // Optionally add task to list without reload
            addTaskToList(data.task);
        } else {
            showToast('Error: ' + JSON.stringify(data.error), 'danger');
        }
    } catch (error) {
        console.error('Error:', error);
        showToast('Failed to create task', 'danger');
    }
}

function addTaskToList(task) {
    const taskList = document.getElementById('task-list');
    const taskCard = `
        <div class="col-md-6 mb-3" id="task-${task.id}">
            <div class="card">
                <div class="card-body">
                    <h5 class="card-title">${task.title}</h5>
                    <p class="card-text">${task.description}</p>
                    <span class="badge badge-priority-${task.priority}">
                        ${task.priority}
                    </span>
                </div>
            </div>
        </div>
    `;
    taskList.insertAdjacentHTML('afterbegin', taskCard);
}
</script>
```

### Example 6: Toast Notifications

**Add to base template:**
```html
<!-- Toast Container -->
<div class="toast-container position-fixed top-0 end-0 p-3" style="z-index: 9999">
    <div id="liveToast" class="toast" role="alert">
        <div class="toast-header">
            <strong class="me-auto">Notification</strong>
            <button type="button" class="btn-close" data-bs-dismiss="toast"></button>
        </div>
        <div class="toast-body" id="toast-message"></div>
    </div>
</div>

<script>
function showToast(message, type = 'info') {
    const toastElement = document.getElementById('liveToast');
    const toastBody = document.getElementById('toast-message');
    
    toastBody.textContent = message;
    toastElement.className = `toast bg-${type} text-white`;
    
    const toast = new bootstrap.Toast(toastElement);
    toast.show();
}
</script>
```

### Example 7: Filter Tasks with AJAX

**HTML Template:**
```html
<select id="status-filter" class="form-control" onchange="filterTasks()">
    <option value="">All Status</option>
    <option value="pending">Pending</option>
    <option value="in_progress">In Progress</option>
    <option value="completed">Completed</option>
</select>

<select id="priority-filter" class="form-control" onchange="filterTasks()">
    <option value="">All Priorities</option>
    <option value="low">Low</option>
    <option value="medium">Medium</option>
    <option value="high">High</option>
</select>

<div id="task-list" class="row"></div>

<script>
async function filterTasks() {
    const status = document.getElementById('status-filter').value;
    const priority = document.getElementById('priority-filter').value;
    
    let url = '/tasks/?';
    if (status) url += `filter=status&filter_by=${status}&`;
    if (priority) url += `filter=priority&filter_by=${priority}`;
    
    try {
        const response = await fetch(url, {
            headers: {
                'Accept': 'application/json'
            }
        });
        
        const data = await response.json();
        
        if (response.ok) {
            renderTasks(data.tasks);
        }
    } catch (error) {
        console.error('Error:', error);
    }
}

function renderTasks(tasks) {
    const taskList = document.getElementById('task-list');
    
    if (tasks.length === 0) {
        taskList.innerHTML = '<p class="text-center">No tasks found</p>';
        return;
    }
    
    let html = '';
    tasks.forEach(task => {
        html += `
            <div class="col-md-6 mb-3" id="task-${task.id}">
                <div class="card ${task.is_completed ? 'task-completed' : ''}">
                    <div class="card-body">
                        <h5 class="card-title">${task.title}</h5>
                        <p class="card-text">${task.description}</p>
                        <span class="badge badge-priority-${task.priority}">
                            ${task.priority}
                        </span>
                        <span class="badge bg-info">${task.status}</span>
                        <div class="btn-group btn-group-sm mt-2">
                            <button onclick="deleteTask(${task.id})" class="btn btn-danger">
                                Delete
                            </button>
                            <button onclick="toggleComplete(${task.id}, ${task.is_completed})" 
                                    class="btn btn-success">
                                ${task.is_completed ? 'Undo' : 'Done'}
                            </button>
                        </div>
                    </div>
                </div>
            </div>
        `;
    });
    
    taskList.innerHTML = html;
}
</script>
```

### Updated Views to Support AJAX

**Update your views to return JSON for AJAX requests:**

```python
from django.http import JsonResponse
import json

@login_required(login_url='login')
def task_list(request):
    tasks = Task.objects.filter(created_by=request.user, is_deleted=False)
    
    # Apply filters
    search_column = request.GET.get('search')
    search_value = request.GET.get('search_by')
    filter_column = request.GET.get('filter')
    filter_value = request.GET.get('filter_by')
    
    if search_column and search_value:
        if search_column == 'all':
            tasks = tasks.filter(
                Q(title__icontains=search_value) | 
                Q(description__icontains=search_value)
            )
    
    if filter_column and filter_value:
        tasks = tasks.filter(**{filter_column: filter_value})
    
    # Return JSON for AJAX requests
    if request.headers.get('Accept') == 'application/json':
        tasks_data = [{
            'id': task.id,
            'title': task.title,
            'description': task.description,
            'priority': task.priority,
            'status': task.status,
            'is_completed': task.is_completed,
            'due_date': task.due_date.isoformat() if task.due_date else None,
        } for task in tasks]
        return JsonResponse({'tasks': tasks_data})
    
    # Return HTML template for regular requests
    context = {
        'tasks': tasks,
        'search_column': search_column,
        'search_value': search_value,
        'filter_column': filter_column,
        'filter_value': filter_value,
    }
    return render(request, 'tasks/task_list.html', context)

@login_required(login_url='login')
def task_toggle_complete(request, pk):
    task = get_object_or_404(Task, pk=pk, created_by=request.user, is_deleted=False)
    
    if task.is_completed:
        task.mark_incomplete()
        message = 'Task marked as incomplete'
    else:
        task.mark_complete()
        message = 'Task marked as complete!'
    
    # Return JSON for AJAX requests
    if request.headers.get('Accept') == 'application/json':
        return JsonResponse({
            'success': True,
            'message': message,
            'is_completed': task.is_completed
        })
    
    # Redirect for regular requests
    messages.success(request, message)
    return redirect('task_list')
```

### AJAX Best Practices

1. **Always include CSRF token** for POST, PATCH, DELETE requests
2. **Check Accept header** to determine if request wants JSON
3. **Use try-catch** to handle errors gracefully
4. **Show loading states** while request is in progress
5. **Validate on both client and server** side
6. **Use async/await** for cleaner code
7. **Debounce search** to avoid too many requests

---

## Summary

You've learned:

✅ Django template syntax (HTML-based, NOT Jinja2)
✅ Template inheritance
✅ Creating reusable base templates
✅ Working with template variables and tags
✅ Using template filters
✅ Creating and validating forms
✅ ModelForm vs Form
✅ Rendering forms in templates
✅ CSRF protection
✅ Building complete UI

### Next Steps

👉 **Continue to:** [`05_FBV_Complete_Project.md`](05_FBV_Complete_Project.md)

In the next section, we'll:
- Put everything together
- Complete the Task Management System
- Add final touches
- Test the application
- Review the complete FBV implementation

---

## Common Mistakes to Avoid

### Mistake 1: Using {{ variable|safe }} Without Sanitization
**Error**: XSS (Cross-Site Scripting) vulnerability
**Solution**: Only use `|safe` filter on trusted content
```django
{# Good - Auto-escaped #}
{{ user_input }}

{# Bad - Dangerous if user_input contains scripts #}
{{ user_input|safe }}
```

### Mistake 2: Not Using {% load static %} for Static Files
**Error**: Static files don't load
**Solution**: Always load static tag at the top of templates
```django
{% load static %}
<link rel="stylesheet" href="{% static 'css/style.css' %}">
```

### Mistake 3: Forgetting form.is_valid() Check
**Error**: Invalid data saved to database
**Solution**: Always validate forms before saving
```python
# Good
if form.is_valid():
    form.save()

# Bad
form.save()  # Skips validation!
```

### Mistake 4: Not Displaying Form Errors
**Error**: Users don't know why form submission failed
**Solution**: Always display form errors in templates
```django
{% if form.errors %}
    <div class="alert alert-danger">
        {{ form.errors }}
    </div>
{% endif %}
```

### Mistake 5: Using Wrong Template Tag Syntax
**Error**: `TemplateSyntaxError`
**Solution**: Remember the difference between `{{ }}` and `{% %}`
```django
{# Variables - use {{ }} #}
{{ task.title }}

{# Tags - use {% %} #}
{% for task in tasks %}
{% endfor %}
```

---

**Templates and Forms are ready! Let's complete the project! 🎉**
