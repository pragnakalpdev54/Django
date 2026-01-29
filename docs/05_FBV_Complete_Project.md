# Part 5: Function-Based Views - Complete Project

## Project Overview

We've built a complete **Task Management System** using Function-Based Views with:
- ✅ HTML templates (Django Template Language)
- ✅ AJAX for API requests (PATCH, DELETE methods)
- ✅ PostgreSQL database
- ✅ Soft delete functionality
- ✅ RESTful API endpoints
- ✅ Modern responsive UI

### Complete Project Structure

```
taskmanager/
├── manage.py
├── .env                          # Environment variables
├── .gitignore                    # Git ignore file
├── requirements.txt              # Python dependencies
├── README.md                     # Project documentation
├── taskmanager/
│   ├── __init__.py
│   ├── settings.py              # Django settings
│   ├── urls.py                  # Main URL configuration
│   ├── wsgi.py                  # WSGI config
│   └── asgi.py                  # ASGI config
└── tasks/
    ├── __init__.py
    ├── admin.py                 # Admin configuration
    ├── apps.py                  # App configuration
    ├── models.py                # Database models
    ├── views.py                 # View functions
    ├── urls.py                  # App URL patterns
    ├── forms.py                 # Form classes
    ├── migrations/
    │   ├── __init__.py
    │   └── 0001_initial.py
    ├── static/
    │   └── tasks/
    │       ├── css/
    │       │   └── custom.css   # Custom styles
    │       └── js/
    │           └── app.js       # AJAX functions
    └── templates/
        └── tasks/
            ├── base.html                    # Base template
            ├── task_list.html              # Task list with AJAX
            ├── task_detail.html            # Task detail
            ├── task_form.html              # Create/Update form
            ├── task_confirm_delete.html    # Delete confirmation
            ├── task_trash.html             # Trash/Recycle bin
            ├── task_restore_confirm.html   # Restore confirmation
            ├── login.html                  # Login page
            └── register.html               # Registration page
```

---

## Complete Code Files

### 1. Environment Variables (`.env`)

```env
# Database Configuration
DB_NAME=taskmanager_db
DB_USER=taskmanager_user
DB_PASSWORD=your_secure_password_here
DB_HOST=localhost
DB_PORT=5432

# Django Settings
SECRET_KEY=your-secret-key-here-generate-new-one
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
```

### 2. Requirements (`requirements.txt`)

```txt
Django==4.2.7
psycopg2-binary==2.9.9
python-decouple==3.8
Pillow==10.1.0
```

### 3. Git Ignore (`.gitignore`)

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python

# Django
*.log
local_settings.py
db.sqlite3
db.sqlite3-journal
media/
staticfiles/

# Environment
.env
venv/
env/
ENV/

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db
```

### 4. Models (`tasks/models.py`)

```python
from django.db import models
from django.contrib.auth.models import User
from django.utils import timezone

class Task(models.Model):
    """Task model with soft delete functionality"""
    
    PRIORITY_CHOICES = [
        ('low', 'Low'),
        ('medium', 'Medium'),
        ('high', 'High'),
    ]
    
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('in_progress', 'In Progress'),
        ('completed', 'Completed'),
    ]
    
    # Basic fields
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    priority = models.CharField(max_length=10, choices=PRIORITY_CHOICES, default='medium')
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')
    
    # Relationships
    created_by = models.ForeignKey(User, on_delete=models.CASCADE, related_name='tasks')
    
    # Timestamps
    created_at = models.DateTimeField(default=timezone.now)
    updated_at = models.DateTimeField(auto_now=True)
    due_date = models.DateField(null=True, blank=True)
    
    # Status flags
    is_completed = models.BooleanField(default=False)
    
    # Soft delete fields
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        ordering = ['-created_at']
        verbose_name = 'Task'
        verbose_name_plural = 'Tasks'
        indexes = [
            models.Index(fields=['created_by', 'is_deleted']),
            models.Index(fields=['status', 'priority']),
        ]
    
    def __str__(self):
        return self.title
    
    def mark_complete(self):
        """Mark task as completed"""
        self.is_completed = True
        self.status = 'completed'
        self.save()
    
    def mark_incomplete(self):
        """Mark task as incomplete"""
        self.is_completed = False
        self.status = 'pending'
        self.save()
    
    def soft_delete(self):
        """Soft delete the task (mark as deleted without removing from database)"""
        self.is_deleted = True
        self.deleted_at = timezone.now()
        self.save()
    
    def restore(self):
        """Restore a soft-deleted task"""
        self.is_deleted = False
        self.deleted_at = None
        self.save()
```

### 5. Forms (`tasks/forms.py`)

```python
from django import forms
from django.contrib.auth.models import User
from django.contrib.auth.forms import UserCreationForm
from .models import Task

class TaskForm(forms.ModelForm):
    class Meta:
        model = Task
        fields = ['title', 'description', 'priority', 'status', 'due_date']
        widgets = {
            'title': forms.TextInput(attrs={'class': 'form-control', 'placeholder': 'Enter task title'}),
            'description': forms.Textarea(attrs={'class': 'form-control', 'rows': 4, 'placeholder': 'Enter task description'}),
            'priority': forms.Select(attrs={'class': 'form-control'}),
            'status': forms.Select(attrs={'class': 'form-control'}),
            'due_date': forms.DateInput(attrs={'class': 'form-control', 'type': 'date'}),
        }
    
    def clean_title(self):
        title = self.cleaned_data.get('title')
        if len(title) < 3:
            raise forms.ValidationError("Title must be at least 3 characters long")
        return title

class UserRegisterForm(UserCreationForm):
    email = forms.EmailField(required=True, widget=forms.EmailInput(attrs={'class': 'form-control', 'placeholder': 'Email'}))
    
    class Meta:
        model = User
        fields = ['username', 'email', 'password1', 'password2']
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fields['username'].widget.attrs.update({'class': 'form-control', 'placeholder': 'Username'})
        self.fields['password1'].widget.attrs.update({'class': 'form-control', 'placeholder': 'Password'})
        self.fields['password2'].widget.attrs.update({'class': 'form-control', 'placeholder': 'Confirm Password'})
```

### 6. Views (`tasks/views.py`) - With AJAX Support

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.contrib.auth import login, logout, authenticate
from django.contrib import messages
from django.http import JsonResponse
from django.db.models import Q
import json
from .models import Task
from .forms import TaskForm, UserRegisterForm

# Task List View - Supports both HTML and JSON responses
@login_required(login_url='login')
def task_list(request):
    """Display task list with search, filter, and sort capabilities"""
    # Exclude soft-deleted tasks
    tasks = Task.objects.filter(created_by=request.user, is_deleted=False)
    
    # Get parameters
    search_column = request.GET.get('search')
    search_value = request.GET.get('search_by')
    filter_column = request.GET.get('filter')
    filter_value = request.GET.get('filter_by')
    sort_column = request.GET.get('sort')
    sort_order = request.GET.get('sort_order', 'asc')
    
    # Apply search
    if search_column and search_value:
        if search_column == 'title':
            tasks = tasks.filter(title__icontains=search_value)
        elif search_column == 'description':
            tasks = tasks.filter(description__icontains=search_value)
        elif search_column == 'all':
            tasks = tasks.filter(Q(title__icontains=search_value) | Q(description__icontains=search_value))
    
    # Apply filter
    if filter_column and filter_value:
        if filter_column == 'status':
            tasks = tasks.filter(status=filter_value)
        elif filter_column == 'priority':
            tasks = tasks.filter(priority=filter_value)
        elif filter_column == 'is_completed':
            is_completed = filter_value.lower() == 'true'
            tasks = tasks.filter(is_completed=is_completed)
    
    # Apply sorting
    if sort_column:
        if sort_order == 'desc':
            sort_column = f'-{sort_column}'
        allowed_sort_columns = ['title', 'created_at', 'updated_at', 'due_date', 'priority', 'status']
        if sort_column.lstrip('-') in allowed_sort_columns:
            tasks = tasks.order_by(sort_column)
    
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
            'created_at': task.created_at.isoformat(),
        } for task in tasks]
        return JsonResponse({'tasks': tasks_data})
    
    # Return HTML template for regular requests
    context = {
        'tasks': tasks,
        'search_column': search_column,
        'search_value': search_value,
        'filter_column': filter_column,
        'filter_value': filter_value,
        'sort_column': sort_column,
        'sort_order': sort_order,
    }
    return render(request, 'tasks/task_list.html', context)

# Task Detail View
@login_required(login_url='login')
def task_detail(request, pk):
    """Display single task details"""
    task = get_object_or_404(Task, pk=pk, created_by=request.user, is_deleted=False)
    return render(request, 'tasks/task_detail.html', {'task': task})

# Task Create View - Supports both POST and JSON
@login_required(login_url='login')
def task_create(request):
    """Create a new task"""
    if request.method == 'POST':
        # Handle JSON data from AJAX
        if request.content_type == 'application/json':
            try:
                data = json.loads(request.body)
                form = TaskForm(data)
            except json.JSONDecodeError:
                return JsonResponse({'error': 'Invalid JSON'}, status=400)
        else:
            # Handle regular form data
            form = TaskForm(request.POST)
        
        if form.is_valid():
            task = form.save(commit=False)
            task.created_by = request.user
            task.save()
            
            # Return JSON for AJAX requests
            if request.headers.get('Accept') == 'application/json':
                return JsonResponse({
                    'success': True,
                    'message': 'Task created successfully!',
                    'task': {
                        'id': task.id,
                        'title': task.title,
                        'description': task.description,
                        'priority': task.priority,
                        'status': task.status,
                    }
                })
            
            messages.success(request, 'Task created successfully!')
            return redirect('task_list')
        else:
            if request.headers.get('Accept') == 'application/json':
                return JsonResponse({'error': form.errors}, status=400)
    else:
        form = TaskForm()
    
    return render(request, 'tasks/task_form.html', {'form': form, 'action': 'Create'})

# Task Update View - Supports POST and PATCH
@login_required(login_url='login')
def task_update(request, pk):
    """Update task - supports both POST (forms) and PATCH (API)"""
    task = get_object_or_404(Task, pk=pk, created_by=request.user, is_deleted=False)
    
    if request.method in ['POST', 'PATCH']:
        # Handle PATCH with JSON
        if request.method == 'PATCH':
            try:
                data = json.loads(request.body)
            except json.JSONDecodeError:
                return JsonResponse({'error': 'Invalid JSON'}, status=400)
            form = TaskForm(data, instance=task)
        else:
            # Handle POST with form data
            form = TaskForm(request.POST, instance=task)
        
        if form.is_valid():
            form.save()
            
            # JSON response for PATCH
            if request.method == 'PATCH':
                return JsonResponse({
                    'success': True,
                    'message': 'Task updated successfully!',
                    'task': {
                        'id': task.id,
                        'title': task.title,
                        'status': task.status,
                        'priority': task.priority,
                    }
                })
            
            # Redirect for POST
            messages.success(request, 'Task updated successfully!')
            return redirect('task_list')
        else:
            if request.method == 'PATCH':
                return JsonResponse({'error': form.errors}, status=400)
    else:
        form = TaskForm(instance=task)
    
    return render(request, 'tasks/task_form.html', {'form': form, 'action': 'Update', 'task': task})

# Task Delete View - Soft Delete with DELETE method
@login_required(login_url='login')
def task_delete(request, pk):
    """Soft delete a task using DELETE method"""
    task = get_object_or_404(Task, pk=pk, created_by=request.user, is_deleted=False)
    
    if request.method == 'DELETE':
        task.soft_delete()
        
        if request.headers.get('Accept') == 'application/json':
            return JsonResponse({
                'success': True,
                'message': 'Task deleted successfully!',
                'deleted_at': task.deleted_at.isoformat()
            })
        
        messages.success(request, 'Task deleted successfully!')
        return redirect('task_list')
    
    return render(request, 'tasks/task_confirm_delete.html', {'task': task})

# Task Restore View
@login_required(login_url='login')
def task_restore(request, pk):
    """Restore a soft-deleted task"""
    task = get_object_or_404(Task, pk=pk, created_by=request.user, is_deleted=True)
    
    if request.method == 'POST':
        task.restore()
        messages.success(request, f'Task "{task.title}" restored successfully!')
        return redirect('task_list')
    
    return render(request, 'tasks/task_restore_confirm.html', {'task': task})

# Task Trash View
@login_required(login_url='login')
def task_trash(request):
    """View all soft-deleted tasks (trash/recycle bin)"""
    deleted_tasks = Task.objects.filter(
        created_by=request.user,
        is_deleted=True
    ).order_by('-deleted_at')
    
    return render(request, 'tasks/task_trash.html', {'deleted_tasks': deleted_tasks})

# Task Toggle Complete View
@login_required(login_url='login')
def task_toggle_complete(request, pk):
    """Toggle task completion status"""
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

# Authentication Views
def register(request):
    """User registration"""
    if request.user.is_authenticated:
        return redirect('task_list')
    
    if request.method == 'POST':
        form = UserRegisterForm(request.POST)
        if form.is_valid():
            user = form.save()
            username = form.cleaned_data.get('username')
            messages.success(request, f'Account created for {username}!')
            login(request, user)
            return redirect('task_list')
    else:
        form = UserRegisterForm()
    
    return render(request, 'tasks/register.html', {'form': form})

def user_login(request):
    """User login"""
    if request.user.is_authenticated:
        return redirect('task_list')
    
    if request.method == 'POST':
        username = request.POST.get('username')
        password = request.POST.get('password')
        user = authenticate(request, username=username, password=password)
        
        if user is not None:
            login(request, user)
            messages.success(request, f'Welcome back, {username}!')
            next_url = request.GET.get('next', 'task_list')
            return redirect(next_url)
        else:
            messages.error(request, 'Invalid username or password')
    
    return render(request, 'tasks/login.html')

@login_required(login_url='login')
def user_logout(request):
    """User logout"""
    logout(request)
    messages.info(request, 'You have been logged out')
    return redirect('login')
```

### 7. URLs (`tasks/urls.py`)

```python
from django.urls import path
from . import views

urlpatterns = [
    # Task URLs
    path('', views.task_list, name='task_list'),
    path('task/create/', views.task_create, name='task_create'),
    path('task/<int:pk>/', views.task_detail, name='task_detail'),
    path('task/<int:pk>/update/', views.task_update, name='task_update'),
    path('task/<int:pk>/delete/', views.task_delete, name='task_delete'),
    path('task/<int:pk>/toggle/', views.task_toggle_complete, name='task_toggle_complete'),
    
    # Soft delete management
    path('task/<int:pk>/restore/', views.task_restore, name='task_restore'),
    path('trash/', views.task_trash, name='task_trash'),
    
    # Authentication URLs
    path('register/', views.register, name='register'),
    path('login/', views.user_login, name='login'),
    path('logout/', views.user_logout, name='logout'),
]
```

### 8. Main URLs (`taskmanager/urls.py`)

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('tasks.urls')),
]
```

### 9. Settings (`taskmanager/settings.py`)

```python
from pathlib import Path
from decouple import config

# Build paths inside the project
BASE_DIR = Path(__file__).resolve().parent.parent

# Security settings
SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='localhost,127.0.0.1').split(',')

# Application definition
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'tasks',  # Our app
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'taskmanager.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

WSGI_APPLICATION = 'taskmanager.wsgi.application'

# Database - PostgreSQL
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST', default='localhost'),
        'PORT': config('DB_PORT', default='5432'),
    }
}

# Password validation
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]

# Internationalization
LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'
USE_I18N = True
USE_TZ = True

# Static files (CSS, JavaScript, Images)
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Media files
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# Default primary key field type
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

# Login URL
LOGIN_URL = 'login'
LOGIN_REDIRECT_URL = 'task_list'
LOGOUT_REDIRECT_URL = 'login'
```
```

### 10. Admin (`tasks/admin.py`)

```python
from django.contrib import admin
from .models import Task

@admin.register(Task)
class TaskAdmin(admin.ModelAdmin):
    list_display = ['title', 'priority', 'status', 'created_by', 'due_date', 'is_completed', 'created_at']
    list_filter = ['status', 'priority', 'is_completed', 'created_at']
    search_fields = ['title', 'description']
    list_editable = ['status', 'priority']
    date_hierarchy = 'created_at'
    readonly_fields = ['created_at', 'updated_at']
    
    fieldsets = (
        ('Task Information', {
            'fields': ('title', 'description', 'created_by')
        }),
        ('Task Details', {
            'fields': ('priority', 'status', 'due_date', 'is_completed')
        }),
        ('Timestamps', {
            'fields': ('created_at', 'updated_at'),
            'classes': ('collapse',)
        }),
    )
```

---

## Complete HTML Templates

### 11. Base Template (`tasks/templates/tasks/base.html`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Task Manager{% endblock %}</title>
    
    <!-- Bootstrap CSS -->
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
        .badge-priority-high { background-color: #dc3545; }
        .badge-priority-medium { background-color: #ffc107; color: #000; }
        .badge-priority-low { background-color: #28a745; }
        .task-completed { opacity: 0.7; }
    </style>
    {% block extra_css %}{% endblock %}
</head>
<body>
    <!-- Navigation -->
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
                            <a class="nav-link" href="{% url 'task_trash' %}">
                                <i class="bi bi-trash"></i> Trash
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

    <!-- Main Content -->
    <div class="container main-content">
        <!-- Django Messages -->
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

    <!-- Bootstrap JS -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    
    <!-- CSRF Token Setup for AJAX -->
    <script>
        // Get CSRF token from cookie
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
        
        // Toast notification function
        function showToast(message, type = 'info') {
            const toastElement = document.getElementById('liveToast');
            const toastBody = document.getElementById('toast-message');
            
            toastBody.textContent = message;
            toastElement.className = `toast bg-${type} text-white`;
            
            const toast = new bootstrap.Toast(toastElement);
            toast.show();
        }
        
        // Delete task with AJAX
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
                    showToast(data.message, 'success');
                    const taskCard = document.getElementById(`task-${taskId}`);
                    if (taskCard) {
                        taskCard.remove();
                    }
                } else {
                    showToast('Error: ' + data.error, 'danger');
                }
            } catch (error) {
                console.error('Error:', error);
                showToast('Failed to delete task', 'danger');
            }
        }
        
        // Toggle task completion with AJAX
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
                    showToast(data.message, 'success');
                    location.reload(); // Reload to see changes
                } else {
                    showToast('Error: ' + data.error, 'danger');
                }
            } catch (error) {
                console.error('Error:', error);
                showToast('Failed to toggle task', 'danger');
            }
        }
    </script>
    
    {% block extra_js %}{% endblock %}
</body>
</html>
```

### 12. Task List Template (`tasks/templates/tasks/task_list.html`)

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
                        <input type="text" name="search_by" class="form-control" 
                               placeholder="Search tasks..." value="{{ search_value|default:'' }}">
                        <input type="hidden" name="search" value="all">
                    </div>
                    <div class="col-md-3">
                        <select name="filter_by" class="form-control" 
                                onchange="document.querySelector('input[name=filter]').value='status'; this.form.submit();">
                            <option value="">All Status</option>
                            <option value="pending" {% if filter_value == 'pending' %}selected{% endif %}>Pending</option>
                            <option value="in_progress" {% if filter_value == 'in_progress' %}selected{% endif %}>In Progress</option>
                            <option value="completed" {% if filter_value == 'completed' %}selected{% endif %}>Completed</option>
                        </select>
                        <input type="hidden" name="filter" value="status">
                    </div>
                    <div class="col-md-3">
                        <select name="sort" class="form-control" onchange="this.form.submit();">
                            <option value="">Sort By</option>
                            <option value="created_at" {% if sort_column == 'created_at' %}selected{% endif %}>Date Created</option>
                            <option value="due_date" {% if sort_column == 'due_date' %}selected{% endif %}>Due Date</option>
                            <option value="priority" {% if sort_column == 'priority' %}selected{% endif %}>Priority</option>
                        </select>
                        <input type="hidden" name="sort_order" value="{{ sort_order|default:'asc' }}">
                    </div>
                    <div class="col-md-2">
                        <button type="submit" class="btn btn-primary w-100">
                            <i class="bi bi-search"></i> Filter
                        </button>
                    </div>
                </form>

                <!-- Task List -->
                {% if tasks %}
                    <div class="row" id="task-list">
                        {% for task in tasks %}
                            <div class="col-md-6 mb-3" id="task-{{ task.id }}">
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
                                            <button onclick="toggleComplete({{ task.id }}, {{ task.is_completed|lower }})" 
                                                    class="btn btn-success">
                                                <i class="bi bi-check-circle"></i> 
                                                {% if task.is_completed %}Undo{% else %}Done{% endif %}
                                            </button>
                                            <button onclick="deleteTask({{ task.id }})" class="btn btn-danger">
                                                <i class="bi bi-trash"></i>
                                            </button>
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

### 13. Task Trash Template (`tasks/templates/tasks/task_trash.html`)

```html
{% extends 'tasks/base.html' %}

{% block title %}Trash - Task Manager{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-12">
        <div class="card">
            <div class="card-body">
                <h2 class="card-title mb-4">
                    <i class="bi bi-trash"></i> Deleted Tasks
                </h2>
                
                {% if deleted_tasks %}
                    <div class="row">
                        {% for task in deleted_tasks %}
                            <div class="col-md-6 mb-3">
                                <div class="card border-danger">
                                    <div class="card-body">
                                        <h5 class="card-title text-muted">{{ task.title }}</h5>
                                        <p class="card-text">{{ task.description|truncatewords:15 }}</p>
                                        <p class="text-muted small">
                                            Deleted: {{ task.deleted_at|date:"M d, Y H:i" }}
                                        </p>
                                        <div class="btn-group btn-group-sm">
                                            <a href="{% url 'task_restore' task.pk %}" class="btn btn-success">
                                                <i class="bi bi-arrow-counterclockwise"></i> Restore
                                            </a>
                                        </div>
                                    </div>
                                </div>
                            </div>
                        {% endfor %}
                    </div>
                {% else %}
                    <div class="alert alert-info text-center">
                        <i class="bi bi-info-circle"></i> Trash is empty.
                    </div>
                {% endif %}
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

---

## Running the Complete Project

### Step 1: Apply Migrations

```bash
python manage.py makemigrations
python manage.py migrate
```

### Step 2: Create Superuser

```bash
python manage.py createsuperuser
```

### Step 3: Run Server

```bash
python manage.py runserver
```

### Step 4: Test the Application

1. **Register**: Visit `http://127.0.0.1:8000/register/`
2. **Login**: Visit `http://127.0.0.1:8000/login/`
3. **Create Task**: Click "New Task"
4. **View Tasks**: See all your tasks on the dashboard
5. **Filter/Search**: Use filters and search
6. **Update Task**: Click "Edit" on any task
7. **Complete Task**: Click "Done" to mark complete
8. **Delete Task**: Click "Delete" and confirm
9. **Admin**: Visit `http://127.0.0.1:8000/admin/`

---

## Features Implemented

### ✅ User Authentication
- User registration with email
- Login/logout functionality
- Password validation
- Session management
- Protected routes

### ✅ CRUD Operations
- **Create**: Add new tasks with form validation
- **Read**: List all tasks, view task details
- **Update**: Edit existing tasks
- **Delete**: Remove tasks with confirmation

### ✅ Task Management
- Priority levels (Low, Medium, High)
- Status tracking (Pending, In Progress, Completed)
- Due date setting
- Task completion toggle
- User-specific tasks

### ✅ Search & Filter
- Search by title and description
- Filter by status
- Filter by priority
- Combined filtering

### ✅ User Interface
- Responsive design with Bootstrap
- Modern gradient background
- Card-based layout
- Icons from Bootstrap Icons
- Flash messages for feedback
- Intuitive navigation

### ✅ Security
- CSRF protection
- Login required decorators
- User ownership validation
- SQL injection prevention (ORM)
- XSS protection (template escaping)

---

## Testing Checklist

### Authentication Tests
- [ ] Register new user
- [ ] Login with correct credentials
- [ ] Login with wrong credentials
- [ ] Logout
- [ ] Access protected page without login (should redirect)

### Task CRUD Tests
- [ ] Create task with all fields
- [ ] Create task with minimal fields
- [ ] View task list
- [ ] View task detail
- [ ] Update task
- [ ] Delete task
- [ ] Try to access another user's task (should fail)

### Filter/Search Tests
- [ ] Search by title
- [ ] Search by description
- [ ] Filter by status
- [ ] Filter by priority
- [ ] Combine search and filters

### Validation Tests
- [ ] Submit empty form
- [ ] Submit task with short title (< 3 chars)
- [ ] Submit task with past due date
- [ ] Register with existing email

### UI Tests
- [ ] Check responsive design on mobile
- [ ] Verify all links work
- [ ] Check flash messages appear
- [ ] Verify icons display correctly

---

## Common Issues and Solutions

### Issue 1: Template Not Found

**Error**: `TemplateDoesNotExist at /`

**Solution**:
- Ensure templates are in `tasks/templates/tasks/` directory
- Check `INSTALLED_APPS` includes 'tasks'
- Verify template names match view render calls

### Issue 2: Static Files Not Loading

**Error**: CSS/JS not loading

**Solution**:
```python
# In settings.py
STATIC_URL = '/static/'

# In template
{% load static %}
<link rel="stylesheet" href="{% static 'css/style.css' %}">
```

### Issue 3: CSRF Token Missing

**Error**: `CSRF verification failed`

**Solution**:
- Add `{% csrf_token %}` in all POST forms
- Ensure `django.middleware.csrf.CsrfViewMiddleware` in MIDDLEWARE

### Issue 4: Login Required Not Working

**Error**: Can access protected pages without login

**Solution**:
- Add `@login_required(login_url='login')` decorator
- Ensure decorator is above view function

### Issue 5: User Can Access Other Users' Tasks

**Error**: Security issue

**Solution**:
```python
# Always filter by current user
task = get_object_or_404(Task, pk=pk, created_by=request.user)
```

---

## Enhancements You Can Add

### 1. Task Categories/Tags

```python
class Category(models.Model):
    name = models.CharField(max_length=50)
    user = models.ForeignKey(User, on_delete=models.CASCADE)

class Task(models.Model):
    # ... existing fields ...
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True)
```

### 2. Task Attachments

```python
class Task(models.Model):
    # ... existing fields ...
    attachment = models.FileField(upload_to='attachments/', null=True, blank=True)
```

### 3. Task Comments

```python
class Comment(models.Model):
    task = models.ForeignKey(Task, on_delete=models.CASCADE, related_name='comments')
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    text = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```

### 4. Email Notifications

```python
from django.core.mail import send_mail

def task_create(request):
    if form.is_valid():
        task = form.save(commit=False)
        task.created_by = request.user
        task.save()
        
        # Send email
        send_mail(
            'New Task Created',
            f'Task "{task.title}" has been created.',
            'from@example.com',
            [request.user.email],
        )
```

### 5. Task Sharing

```python
class Task(models.Model):
    # ... existing fields ...
    shared_with = models.ManyToManyField(User, related_name='shared_tasks', blank=True)
```

### 6. Dashboard Statistics

```python
def dashboard(request):
    total_tasks = Task.objects.filter(created_by=request.user).count()
    completed_tasks = Task.objects.filter(created_by=request.user, is_completed=True).count()
    pending_tasks = total_tasks - completed_tasks
    
    context = {
        'total_tasks': total_tasks,
        'completed_tasks': completed_tasks,
        'pending_tasks': pending_tasks,
    }
    return render(request, 'tasks/dashboard.html', context)
```

---

## Summary of Function-Based Views

### Advantages of FBV

✅ **Simple and Explicit**: Easy to understand what's happening
✅ **Full Control**: Complete control over request/response
✅ **Flexible**: Can handle any custom logic
✅ **Beginner-Friendly**: Easier to learn and debug
✅ **Straightforward**: One function = one view

### When to Use FBV

- Simple views with unique logic
- Custom workflows
- Learning Django
- Views that don't fit generic patterns
- When you need full control

### FBV Pattern Summary

```python
@login_required
def my_view(request, pk):
    # 1. Get object
    obj = get_object_or_404(Model, pk=pk)
    
    # 2. Handle POST
    if request.method == 'POST':
        form = MyForm(request.POST, instance=obj)
        if form.is_valid():
            form.save()
            messages.success(request, 'Success!')
            return redirect('list_view')
    
    # 3. Handle GET
    else:
        form = MyForm(instance=obj)
    
    # 4. Render template
    return render(request, 'template.html', {'form': form})
```

---

## Next Steps

👉 **Continue to:** [`06_CBV_Introduction.md`](06_CBV_Introduction.md)

Now that you've mastered Function-Based Views, let's learn Class-Based Views and see how we can refactor the same project using CBV!

In the next section, we'll:
- Understand Class-Based Views
- Learn about generic views
- Refactor our Task Manager to use CBV
- Compare FBV vs CBV approaches

---

**Congratulations! You've completed a full Django project with FBV! 🎉**
