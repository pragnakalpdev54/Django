# Part 3: Function-Based Views - Views & URLs

## Understanding Views

Views are Python functions (or classes) that receive web requests and return web responses. They contain the business logic of your application.

### Request-Response Cycle

```
User Browser → URL → View Function → Template → HTML Response → User Browser
```

### Basic View Structure

```python
from django.http import HttpResponse

def my_view(request):
    # Process request
    # Query database
    # Perform logic
    return HttpResponse("Hello, World!")
```

---

## Creating Function-Based Views Step-by-Step

Let's build our views progressively, starting simple and adding features one by one. This approach helps you understand each component.

### Step 1: Your First Simple View

Let's start with the simplest possible view. Open `tasks/views.py` and create:

```python
from django.shortcuts import render
from .models import Task

def task_list(request):
    """Display all tasks"""
    tasks = Task.objects.all()
    return render(request, 'tasks/task_list.html', {'tasks': tasks})
```

**What's happening here?**
1. `def task_list(request)` - Every view takes `request` as first parameter
2. `Task.objects.all()` - Get all tasks from database
3. `render()` - Combines template with data and returns HTML
4. `{'tasks': tasks}` - Context dictionary passed to template

### Step 2: Add User Filtering

Now let's show only the current user's tasks:

```python
from django.shortcuts import render
from .models import Task

def task_list(request):
    """Display tasks for the logged-in user"""
    # Filter tasks by current user
    tasks = Task.objects.filter(created_by=request.user)
    return render(request, 'tasks/task_list.html', {'tasks': tasks})
```

**New concept:**
- `request.user` - The currently logged-in user
- `filter(created_by=request.user)` - Only get tasks created by this user

### Step 3: Add Login Protection

Protect the view so only logged-in users can access it:

```python
from django.shortcuts import render
from django.contrib.auth.decorators import login_required
from .models import Task

@login_required(login_url='login')
def task_list(request):
    """Display tasks for the logged-in user"""
    tasks = Task.objects.filter(created_by=request.user)
    return render(request, 'tasks/task_list.html', {'tasks': tasks})
```

**New concept:**
- `@login_required` - Decorator that checks if user is logged in
- If not logged in, redirects to `login_url`
- Must be placed right above the function

### Step 4: Add Search Functionality

Let's add the ability to search tasks with a structured approach:

```python
from django.shortcuts import render
from django.contrib.auth.decorators import login_required
from django.db.models import Q
from .models import Task

@login_required(login_url='login')
def task_list(request):
    """Display tasks with search functionality"""
    # Start with user's tasks
    tasks = Task.objects.filter(created_by=request.user)
    
    # Get search parameters from URL (e.g., ?search=title&search_by=django)
    search_column = request.GET.get('search')  # Which column to search
    search_value = request.GET.get('search_by')  # What to search for
    
    # If both search parameters exist, apply search
    if search_column and search_value:
        if search_column == 'title':
            tasks = tasks.filter(title__icontains=search_value)
        elif search_column == 'description':
            tasks = tasks.filter(description__icontains=search_value)
        elif search_column == 'all':
            # Search in both title and description
            tasks = tasks.filter(
                Q(title__icontains=search_value) | 
                Q(description__icontains=search_value)
            )
    
    context = {
        'tasks': tasks,
        'search_column': search_column,
        'search_value': search_value,
    }
    return render(request, 'tasks/task_list.html', context)
```

**New concepts:**
- `request.GET.get('search')` - Get column name to search in
- `request.GET.get('search_by')` - Get value to search for
- `Q()` - Allows complex queries with OR/AND logic
- `title__icontains` - Case-insensitive search in title
- `|` - OR operator (search in title OR description)

**URL Examples:**
- Search in title: `?search=title&search_by=django`
- Search in description: `?search=description&search_by=meeting`
- Search in all: `?search=all&search_by=important`

### Step 5: Add Filtering and Sorting

Now add filtering and sorting with a structured approach:

```python
from django.shortcuts import render
from django.contrib.auth.decorators import login_required
from django.db.models import Q
from .models import Task

@login_required(login_url='login')
def task_list(request):
    """Display tasks with search, filter, and sort"""
    # Start with user's tasks
    tasks = Task.objects.filter(created_by=request.user)
    
    # Get search parameters (e.g., ?search=title&search_by=django)
    search_column = request.GET.get('search')
    search_value = request.GET.get('search_by')
    
    # Get filter parameters (e.g., ?filter=status&filter_by=pending)
    filter_column = request.GET.get('filter')
    filter_value = request.GET.get('filter_by')
    
    # Get sort parameters (e.g., ?sort=created_at&sort_order=desc)
    sort_column = request.GET.get('sort')
    sort_order = request.GET.get('sort_order', 'asc')  # Default to ascending
    
    # Apply search if provided
    if search_column and search_value:
        if search_column == 'title':
            tasks = tasks.filter(title__icontains=search_value)
        elif search_column == 'description':
            tasks = tasks.filter(description__icontains=search_value)
        elif search_column == 'all':
            tasks = tasks.filter(
                Q(title__icontains=search_value) | 
                Q(description__icontains=search_value)
            )
    
    # Apply filter if provided
    if filter_column and filter_value:
        if filter_column == 'status':
            tasks = tasks.filter(status=filter_value)
        elif filter_column == 'priority':
            tasks = tasks.filter(priority=filter_value)
        elif filter_column == 'is_completed':
            # Convert string to boolean
            is_completed = filter_value.lower() == 'true'
            tasks = tasks.filter(is_completed=is_completed)
    
    # Apply sorting if provided
    if sort_column:
        # Add '-' prefix for descending order
        if sort_order == 'desc':
            sort_column = f'-{sort_column}'
        
        # Validate sort column to prevent errors
        allowed_sort_columns = ['title', 'created_at', 'updated_at', 'due_date', 'priority', 'status']
        if sort_column.lstrip('-') in allowed_sort_columns:
            tasks = tasks.order_by(sort_column)
    
    # Prepare context for template
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
```

**What's happening:**

**1. Search Pattern:**
- `?search=title&search_by=django` - Search in title column for "django"
- `?search=all&search_by=meeting` - Search in all columns for "meeting"

**2. Filter Pattern:**
- `?filter=status&filter_by=pending` - Filter by status = pending
- `?filter=priority&filter_by=high` - Filter by priority = high
- `?filter=is_completed&filter_by=true` - Filter completed tasks

**3. Sort Pattern:**
- `?sort=created_at&sort_order=desc` - Sort by created_at descending
- `?sort=title&sort_order=asc` - Sort by title ascending
- `?sort=due_date&sort_order=desc` - Sort by due_date descending

**4. Combining Parameters:**
- `?search=title&search_by=django&filter=status&filter_by=pending&sort=created_at&sort_order=desc`
- All parameters work together (cumulative)

**Security Note:**
- We validate `sort_column` against allowed columns to prevent SQL injection
- Always validate user input before using in queries

**✅ Complete task_list view with search, filter, and sort is ready!**

---

### Step 6: Create Task Detail View

Now let's create a view to show a single task:

```python
from django.shortcuts import render, get_object_or_404
from django.contrib.auth.decorators import login_required
from .models import Task

@login_required(login_url='login')
def task_detail(request, pk):
    """Display a single task"""
    # Get task by ID (pk) and ensure user owns it
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    return render(request, 'tasks/task_detail.html', {'task': task})
```

**New concepts:**
- `pk` - Primary key (ID) of the task, comes from URL
- `get_object_or_404()` - Gets object or shows 404 error if not found
- Security: `created_by=request.user` ensures user can only view their own tasks

**✅ Task detail view is ready!**

---

## Creating Django Forms

Before we create views that handle forms, we need to create the form classes. Django forms handle user input, validation, and data cleaning.

### Step 1: Create forms.py File

Create a new file `tasks/forms.py`:

```python
from django import forms
from .models import Task

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
    
    def clean_title(self):
        """Validate title length"""
        title = self.cleaned_data.get('title')
        if len(title) < 3:
            raise forms.ValidationError("Title must be at least 3 characters long")
        return title
    
    def clean_due_date(self):
        """Validate due date is not in the past"""
        due_date = self.cleaned_data.get('due_date')
        if due_date:
            from django.utils import timezone
            if due_date < timezone.now().date():
                raise forms.ValidationError("Due date cannot be in the past")
        return due_date
```

**Understanding ModelForm:**

1. **ModelForm** - Automatically creates form from model
   - Saves you from writing repetitive code
   - Handles model saving automatically
   - Best for CRUD operations

2. **Meta class** - Configuration for the form
   - `model = Task` - Which model to use
   - `fields = [...]` - Which fields to include
   - `widgets = {...}` - Customize HTML input elements

3. **Widgets** - Control how fields are rendered
   - Add CSS classes for styling
   - Set HTML attributes (placeholder, type, etc.)
   - Customize input appearance

4. **Validation Methods** - Custom validation logic
   - `clean_fieldname()` - Validate specific field
   - Raise `ValidationError` if invalid
   - Return cleaned data if valid

**Why use forms?**
- ✅ Automatic validation
- ✅ Security (CSRF protection)
- ✅ Clean data handling
- ✅ Easy rendering in templates
- ✅ Error messages

**✅ TaskForm is ready! Now we can use it in views.**

---

### Step 7: Create Task Creation View (Understanding GET vs POST)

This view handles both displaying a form (GET) and processing it (POST):

```python
from django.shortcuts import render, redirect
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from .forms import TaskForm

@login_required(login_url='login')
def task_create(request):
    """Create a new task"""
    
    # Check if form was submitted (POST request)
    if request.method == 'POST':
        # Create form with submitted data
        form = TaskForm(request.POST)
        
        # Validate form
        if form.is_valid():
            # Save form but don't commit to database yet
            task = form.save(commit=False)
            
            # Set the creator
            task.created_by = request.user
            
            # Now save to database
            task.save()
            
            # Show success message
            messages.success(request, 'Task created successfully!')
            
            # Redirect to task list
            return redirect('task_list')
    
    else:
        # GET request - just show empty form
        form = TaskForm()
    
    # Render template with form
    return render(request, 'tasks/task_form.html', {
        'form': form,
        'action': 'Create'
    })
```

**Understanding the flow:**

**When user visits the page (GET):**
1. `request.method` is 'GET'
2. Create empty form: `form = TaskForm()`
3. Show form to user

**When user submits form (POST):**
1. `request.method` is 'POST'
2. Create form with data: `TaskForm(request.POST)`
3. Validate: `form.is_valid()`
4. Save: `form.save()`
5. Redirect to prevent duplicate submission

**Why `commit=False`?**
- We need to set `created_by` before saving
- `commit=False` creates the object but doesn't save to database yet
- We set `created_by`, then call `save()`

**✅ Task creation view is ready!**

---

### Step 8: Create Task Update View (Using PATCH Method)

For updates, we'll use the **PATCH** HTTP method, which is the standard for partial updates:

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.views.decorators.http import require_http_methods
from django.contrib import messages
from django.http import JsonResponse
from .models import Task
from .forms import TaskForm
import json

@login_required(login_url='login')
@require_http_methods(["GET", "PATCH"])
def task_update(request, pk):
    """Update an existing task using PATCH method"""
    
    # Get the task to update (ensure user owns it)
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    
    # Handle PATCH request for updates
    if request.method == 'PATCH':
        # Parse JSON data from PATCH request
        try:
            data = json.loads(request.body)
        except json.JSONDecodeError:
            return JsonResponse({'error': 'Invalid JSON'}, status=400)
        
        # Create form with PATCH data
        form = TaskForm(data, instance=task)
        
        if form.is_valid():
            # Save updates
            form.save()
            
            # Return JSON response for API calls
            if request.headers.get('Accept') == 'application/json':
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
            
            messages.success(request, 'Task updated successfully!')
            return redirect('task_list')
        else:
            return JsonResponse({'error': form.errors}, status=400)
    
    # GET request - show form pre-filled with task data
    form = TaskForm(instance=task)
    return render(request, 'tasks/task_form.html', {
        'form': form,
        'action': 'Update',
        'task': task
    })
```

**Understanding PATCH Method:**

**Why PATCH instead of POST?**
- **POST** = Create new resource
- **PATCH** = Partially update existing resource
- **PUT** = Replace entire resource
- PATCH is RESTful and semantically correct for updates

**Key concepts:**
1. `@require_http_methods(["GET", "PATCH"])` - Only allow GET and PATCH
2. `request.method == 'PATCH'` - Check for PATCH request
3. `json.loads(request.body)` - Parse JSON data from request body
4. Return JSON response for API compatibility

**Alternative: Support Both POST and PATCH**

For backward compatibility, you can support both:

```python
@login_required(login_url='login')
def task_update(request, pk):
    """Update task - supports both POST (forms) and PATCH (API)"""
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    
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
                    'message': 'Task updated successfully!'
                })
            
            # Redirect for POST
            messages.success(request, 'Task updated successfully!')
            return redirect('task_list')
    
    # GET - show form
    form = TaskForm(instance=task)
    return render(request, 'tasks/task_form.html', {
        'form': form,
        'action': 'Update',
        'task': task
    })
```

**✅ Task update view with PATCH method is ready!**

---

### Step 9: Create Task Delete View (Using DELETE Method & Soft Delete)

**Understanding Soft Delete:**
- **Hard Delete**: Permanently removes data from database
- **Soft Delete**: Marks data as deleted but keeps it in database
- Benefits: Can restore deleted items, maintain data history, audit trail

#### Option 1: Soft Delete (Recommended)

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.views.decorators.http import require_http_methods
from django.contrib import messages
from django.http import JsonResponse
from .models import Task

@login_required(login_url='login')
@require_http_methods(["GET", "DELETE"])
def task_delete(request, pk):
    """Soft delete a task using DELETE method"""
    
    # Get the task (ensure user owns it and not already deleted)
    task = get_object_or_404(Task, pk=pk, created_by=request.user, is_deleted=False)
    
    # Handle DELETE request
    if request.method == 'DELETE':
        # Soft delete the task
        task.soft_delete()
        
        # Return JSON response for API calls
        if request.headers.get('Accept') == 'application/json':
            return JsonResponse({
                'success': True,
                'message': 'Task deleted successfully!',
                'deleted_at': task.deleted_at.isoformat()
            })
        
        messages.success(request, 'Task deleted successfully!')
        return redirect('task_list')
    
    # GET request - show confirmation page
    return render(request, 'tasks/task_confirm_delete.html', {'task': task})
```

**Key concepts:**
1. `@require_http_methods(["GET", "DELETE"])` - Only allow GET and DELETE
2. `task.soft_delete()` - Marks task as deleted (sets `is_deleted=True`)
3. Task remains in database for recovery
4. `is_deleted=False` in query ensures we don't try to delete already deleted tasks

#### Option 2: Hard Delete with DELETE Method

```python
@login_required(login_url='login')
@require_http_methods(["GET", "DELETE"])
def task_hard_delete(request, pk):
    """Permanently delete a task using DELETE method"""
    
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    
    if request.method == 'DELETE':
        task_title = task.title  # Save for message
        task.delete()  # Permanently remove from database
        
        if request.headers.get('Accept') == 'application/json':
            return JsonResponse({
                'success': True,
                'message': f'Task "{task_title}" permanently deleted!'
            })
        
        messages.success(request, f'Task "{task_title}" permanently deleted!')
        return redirect('task_list')
    
    return render(request, 'tasks/task_confirm_delete.html', {'task': task})
```

#### Restore Deleted Task View

```python
@login_required(login_url='login')
def task_restore(request, pk):
    """Restore a soft-deleted task"""
    
    # Get soft-deleted task
    task = get_object_or_404(Task, pk=pk, created_by=request.user, is_deleted=True)
    
    if request.method == 'POST':
        task.restore()
        messages.success(request, f'Task "{task.title}" restored successfully!')
        return redirect('task_list')
    
    return render(request, 'tasks/task_restore_confirm.html', {'task': task})
```

#### View Deleted Tasks

```python
@login_required(login_url='login')
def task_trash(request):
    """View all soft-deleted tasks (trash/recycle bin)"""
    
    deleted_tasks = Task.objects.filter(
        created_by=request.user,
        is_deleted=True
    ).order_by('-deleted_at')
    
    return render(request, 'tasks/task_trash.html', {
        'deleted_tasks': deleted_tasks
    })
```

**The flow:**
1. User clicks delete (GET request) → Show confirmation
2. User confirms (DELETE request) → Soft delete task
3. Task moved to trash (can be restored)
4. User can view trash and restore tasks

**Update task_list to exclude deleted tasks:**

```python
@login_required(login_url='login')
def task_list(request):
    # Exclude soft-deleted tasks
    tasks = Task.objects.filter(created_by=request.user, is_deleted=False)
    # ... rest of the code
```

**✅ Task delete view with DELETE method and soft delete is ready!**

---

### Step 10: Create Toggle Complete View

A simple view to mark tasks as complete/incomplete:

```python
from django.shortcuts import redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from .models import Task

@login_required(login_url='login')
def task_toggle_complete(request, pk):
    """Toggle task completion status"""
    
    # Get the task
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    
    # Check current status and toggle
    if task.is_completed:
        task.mark_incomplete()  # Using model method
        messages.info(request, 'Task marked as incomplete')
    else:
        task.mark_complete()  # Using model method
        messages.success(request, 'Task marked as complete!')
    
    # Redirect back to task list
    return redirect('task_list')
```

**Note:**
- We use model methods `mark_complete()` and `mark_incomplete()`
- This keeps business logic in the model (good practice)
- No template needed - just redirect after action

**✅ Task toggle view is ready!**

---

### Complete views.py File

Now that you understand each view, here's the complete file:

```python
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.contrib import messages
from django.db.models import Q
from .models import Task
from .forms import TaskForm

@login_required(login_url='login')
def task_list(request):
    tasks = Task.objects.filter(created_by=request.user)
    
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

@login_required(login_url='login')
def task_detail(request, pk):
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    return render(request, 'tasks/task_detail.html', {'task': task})

@login_required(login_url='login')
def task_create(request):
    if request.method == 'POST':
        form = TaskForm(request.POST)
        if form.is_valid():
            task = form.save(commit=False)
            task.created_by = request.user
            task.save()
            messages.success(request, 'Task created successfully!')
            return redirect('task_list')
    else:
        form = TaskForm()
    return render(request, 'tasks/task_form.html', {'form': form, 'action': 'Create'})

@login_required(login_url='login')
def task_update(request, pk):
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    if request.method == 'POST':
        form = TaskForm(request.POST, instance=task)
        if form.is_valid():
            form.save()
            messages.success(request, 'Task updated successfully!')
            return redirect('task_list')
    else:
        form = TaskForm(instance=task)
    return render(request, 'tasks/task_form.html', {'form': form, 'action': 'Update', 'task': task})

@login_required(login_url='login')
def task_delete(request, pk):
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    if request.method == 'POST':
        task.delete()
        messages.success(request, 'Task deleted successfully!')
        return redirect('task_list')
    return render(request, 'tasks/task_confirm_delete.html', {'task': task})

@login_required(login_url='login')
def task_toggle_complete(request, pk):
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    if task.is_completed:
        task.mark_incomplete()
        messages.info(request, 'Task marked as incomplete')
    else:
        task.mark_complete()
        messages.success(request, 'Task marked as complete!')
    return redirect('task_list')
```

### Understanding View Components

**1. Decorators**
```python
@login_required(login_url='login')
```
- Ensures user is authenticated
- Redirects to login page if not
- Must be imported: `from django.contrib.auth.decorators import login_required`

**2. Request Object**
```python
def my_view(request):
    # request.method - HTTP method (GET, POST, etc.)
    # request.GET - URL parameters
    # request.POST - Form data
    # request.user - Current user
    # request.FILES - Uploaded files
```

**3. get_object_or_404**
```python
task = get_object_or_404(Task, pk=pk, created_by=request.user)
```
- Gets object or returns 404 error
- Better than `Task.objects.get()` which raises exception
- Automatically handles DoesNotExist

**4. Messages Framework**
```python
from django.contrib import messages

messages.success(request, 'Task created successfully!')
messages.error(request, 'Something went wrong')
messages.warning(request, 'Be careful!')
messages.info(request, 'FYI: Something happened')
```

**5. Redirect**
```python
return redirect('task_list')  # By URL name
return redirect('/tasks/')     # By path
return redirect(task)          # By object (needs get_absolute_url)
```

**6. Render**
```python
return render(request, 'template.html', context)
```
- Renders template with context data
- Returns HttpResponse

---

## URL Configuration

URLs map web addresses to view functions.

### Step 1: Create App URLs

Create `tasks/urls.py`:

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
]
```

### Step 2: Include App URLs in Project

Open `taskmanager/urls.py`:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('tasks.urls')),
]
```

### URL Patterns Explained

**Basic Pattern**
```python
path('tasks/', views.task_list, name='task_list')
```
- First argument: URL pattern
- Second argument: View function
- Third argument: URL name (for reverse lookup)

**URL Parameters**
```python
path('task/<int:pk>/', views.task_detail, name='task_detail')
```
- `<int:pk>`: Captures integer as `pk` parameter
- Passed to view: `def task_detail(request, pk)`

**Parameter Types**
```python
<int:id>      # Integer
<str:slug>    # String (no slashes)
<slug:slug>   # Slug (letters, numbers, hyphens, underscores)
<uuid:id>     # UUID
<path:path>   # Any string (including slashes)
```

**Multiple Parameters**
```python
path('user/<int:user_id>/task/<int:task_id>/', views.user_task_detail)
```

### URL Naming and Reverse Lookup

**Why name URLs?**
- Avoid hardcoding URLs
- Easy to change URL structure
- Use in templates and views

**In Views:**
```python
from django.urls import reverse

url = reverse('task_detail', kwargs={'pk': 1})
# Returns: '/task/1/'

return redirect('task_list')
```

**In Templates:**
```html
<a href="{% url 'task_list' %}">All Tasks</a>
<a href="{% url 'task_detail' task.pk %}">View Task</a>
<a href="{% url 'task_update' pk=task.pk %}">Edit</a>
```

---

## HTTP Methods in Views

### GET vs POST

**GET Request**
- Retrieve data
- Parameters in URL
- Idempotent (safe to repeat)
- Used for: viewing, searching, filtering

**POST Request**
- Submit data
- Parameters in request body
- Not idempotent
- Used for: creating, updating, deleting

### Handling Different Methods

```python
def my_view(request):
    if request.method == 'POST':
        # Handle form submission
        form = MyForm(request.POST)
        if form.is_valid():
            form.save()
            return redirect('success')
    else:
        # Display form
        form = MyForm()
    
    return render(request, 'form.html', {'form': form})
```

### Method-Based Logic

```python
def task_operations(request, pk):
    task = get_object_or_404(Task, pk=pk)
    
    if request.method == 'GET':
        # Display task
        return render(request, 'task_detail.html', {'task': task})
    
    elif request.method == 'POST':
        # Update task
        task.title = request.POST.get('title')
        task.save()
        return redirect('task_list')
    
    elif request.method == 'DELETE':
        # Delete task
        task.delete()
        return HttpResponse(status=204)
```

---

## Working with Request Data

### GET Parameters

**Example 1: Search with structured parameters**
```python
def search_tasks(request):
    # URL: /search/?search=title&search_by=django&filter=status&filter_by=pending
    
    tasks = Task.objects.all()
    
    # Get search parameters
    search_column = request.GET.get('search')
    search_value = request.GET.get('search_by')
    
    # Get filter parameters
    filter_column = request.GET.get('filter')
    filter_value = request.GET.get('filter_by')
    
    # Apply search
    if search_column and search_value:
        if search_column == 'title':
            tasks = tasks.filter(title__icontains=search_value)
        elif search_column == 'description':
            tasks = tasks.filter(description__icontains=search_value)
    
    # Apply filter
    if filter_column and filter_value:
        tasks = tasks.filter(**{filter_column: filter_value})
    
    return render(request, 'search.html', {'tasks': tasks})
```

**Example 2: Multiple filters**
```python
def advanced_search(request):
    # URL: /search/?search=all&search_by=django&filter=status&filter_by=pending&sort=created_at&sort_order=desc
    
    tasks = Task.objects.all()
    
    # Search
    search_col = request.GET.get('search')
    search_val = request.GET.get('search_by')
    if search_col == 'all' and search_val:
        tasks = tasks.filter(
            Q(title__icontains=search_val) | 
            Q(description__icontains=search_val)
        )
    
    # Filter
    filter_col = request.GET.get('filter')
    filter_val = request.GET.get('filter_by')
    if filter_col and filter_val:
        tasks = tasks.filter(**{filter_col: filter_val})
    
    # Sort
    sort_col = request.GET.get('sort')
    sort_order = request.GET.get('sort_order', 'asc')
    if sort_col:
        order_by = f'-{sort_col}' if sort_order == 'desc' else sort_col
        tasks = tasks.order_by(order_by)
    
    return render(request, 'search.html', {'tasks': tasks})
```

### POST Data

```python
def create_task(request):
    if request.method == 'POST':
        title = request.POST.get('title')
        description = request.POST.get('description')
        priority = request.POST.get('priority', 'medium')
        
        task = Task.objects.create(
            title=title,
            description=description,
            priority=priority,
            created_by=request.user
        )
        
        return redirect('task_detail', pk=task.pk)
    
    return render(request, 'create_task.html')
```

### File Uploads

```python
def upload_avatar(request):
    if request.method == 'POST':
        uploaded_file = request.FILES.get('avatar')
        
        if uploaded_file:
            # Save file
            profile = request.user.profile
            profile.avatar = uploaded_file
            profile.save()
            
            messages.success(request, 'Avatar uploaded!')
        
        return redirect('profile')
    
    return render(request, 'upload.html')
```

---

## Authentication Views

### User Registration

```python
from django.contrib.auth import login
from django.contrib.auth.forms import UserCreationForm

def register(request):
    if request.user.is_authenticated:
        return redirect('task_list')
    
    if request.method == 'POST':
        form = UserCreationForm(request.POST)
        if form.is_valid():
            user = form.save()
            username = form.cleaned_data.get('username')
            messages.success(request, f'Account created for {username}!')
            login(request, user)  # Auto-login after registration
            return redirect('task_list')
    else:
        form = UserCreationForm()
    
    return render(request, 'tasks/register.html', {'form': form})
```

### User Login

```python
from django.contrib.auth import authenticate, login

def user_login(request):
    if request.user.is_authenticated:
        return redirect('task_list')
    
    if request.method == 'POST':
        username = request.POST.get('username')
        password = request.POST.get('password')
        user = authenticate(request, username=username, password=password)
        
        if user is not None:
            login(request, user)
            messages.success(request, f'Welcome back, {username}!')
            
            # Redirect to 'next' parameter or default
            next_url = request.GET.get('next', 'task_list')
            return redirect(next_url)
        else:
            messages.error(request, 'Invalid username or password')
    
    return render(request, 'tasks/login.html')
```

### User Logout

```python
from django.contrib.auth import logout

@login_required
def user_logout(request):
    logout(request)
    messages.info(request, 'You have been logged out')
    return redirect('login')
```

### Update URLs for Authentication

Add to `tasks/urls.py`:

```python
urlpatterns = [
    # ... existing task URLs ...
    
    # Authentication URLs
    path('register/', views.register, name='register'),
    path('login/', views.user_login, name='login'),
    path('logout/', views.user_logout, name='logout'),
]
```

---

## Advanced View Patterns

### Pagination

```python
from django.core.paginator import Paginator

def task_list(request):
    tasks = Task.objects.filter(created_by=request.user)
    
    # Create paginator (10 items per page)
    paginator = Paginator(tasks, 10)
    
    # Get page number from URL
    page_number = request.GET.get('page', 1)
    
    # Get page object
    page_obj = paginator.get_page(page_number)
    
    return render(request, 'task_list.html', {'page_obj': page_obj})
```

**In template:**
```html
{% for task in page_obj %}
    <!-- Display task -->
{% endfor %}

<!-- Pagination controls -->
<div class="pagination">
    {% if page_obj.has_previous %}
        <a href="?page=1">First</a>
        <a href="?page={{ page_obj.previous_page_number }}">Previous</a>
    {% endif %}
    
    <span>Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}</span>
    
    {% if page_obj.has_next %}
        <a href="?page={{ page_obj.next_page_number }}">Next</a>
        <a href="?page={{ page_obj.paginator.num_pages }}">Last</a>
    {% endif %}
</div>
```

### AJAX Views (JSON Response)

```python
from django.http import JsonResponse

def task_status_api(request, pk):
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    
    if request.method == 'POST':
        new_status = request.POST.get('status')
        task.status = new_status
        task.save()
        
        return JsonResponse({
            'success': True,
            'message': 'Status updated',
            'task': {
                'id': task.id,
                'title': task.title,
                'status': task.status,
            }
        })
    
    return JsonResponse({
        'id': task.id,
        'title': task.title,
        'status': task.status,
        'priority': task.priority,
    })
```

### Custom 404 and 500 Views

Create `tasks/views.py`:

```python
def custom_404(request, exception):
    return render(request, '404.html', status=404)

def custom_500(request):
    return render(request, '500.html', status=500)
```

In `taskmanager/urls.py`:

```python
handler404 = 'tasks.views.custom_404'
handler500 = 'tasks.views.custom_500'
```

---

## View Best Practices

### 1. Keep Views Thin

Move business logic to models or separate functions:

```python
# Bad
def task_create(request):
    if request.method == 'POST':
        task = Task()
        task.title = request.POST.get('title')
        task.description = request.POST.get('description')
        # ... lots of logic ...
        task.save()

# Good
def task_create(request):
    if request.method == 'POST':
        form = TaskForm(request.POST)
        if form.is_valid():
            task = form.save(commit=False)
            task.created_by = request.user
            task.save()
            return redirect('task_list')
```

### 2. Use get_object_or_404

```python
# Bad
try:
    task = Task.objects.get(pk=pk)
except Task.DoesNotExist:
    return HttpResponse('Not found', status=404)

# Good
task = get_object_or_404(Task, pk=pk)
```

### 3. Always Validate User Permissions

```python
# Ensure user owns the task
task = get_object_or_404(Task, pk=pk, created_by=request.user)
```

### 4. Use Messages for Feedback

```python
messages.success(request, 'Task created successfully!')
messages.error(request, 'Failed to create task')
```

### 5. Redirect After POST

```python
# Prevent duplicate submissions
if request.method == 'POST':
    # Process form
    return redirect('success_page')
```

### 6. Handle Both GET and POST

```python
def my_view(request):
    if request.method == 'POST':
        # Handle form submission
        pass
    else:
        # Display form
        pass
```

---

## URL Organization

### Namespacing URLs

**In app urls.py:**
```python
app_name = 'tasks'

urlpatterns = [
    path('', views.task_list, name='list'),
    path('<int:pk>/', views.task_detail, name='detail'),
]
```

**Usage:**
```python
# In views
return redirect('tasks:list')

# In templates
{% url 'tasks:detail' task.pk %}
```

### URL Includes with Prefix

```python
# taskmanager/urls.py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('tasks/', include('tasks.urls')),
    path('api/', include('api.urls')),
]
```

---

## Summary

You've learned:

✅ Creating function-based views
✅ Handling GET and POST requests
✅ URL configuration and routing
✅ URL parameters and naming
✅ Authentication decorators
✅ Request and response objects
✅ Messages framework
✅ Pagination
✅ JSON responses
✅ View best practices

### Next Steps

👉 **Continue to:** [`04_FBV_Templates_and_Forms.md`](04_FBV_Templates_and_Forms.md)

In the next section, we'll:
- Create Django templates
- Use template inheritance
- Work with forms
- Display data dynamically
- Build the complete UI

---

## Common Mistakes to Avoid

### Mistake 1: Forgetting @login_required Decorator
**Error**: Unauthenticated users can access protected views
**Solution**: Always add `@login_required` to views that need authentication
```python
from django.contrib.auth.decorators import login_required

@login_required(login_url='login')
def task_list(request):
    # Your code
```

### Mistake 2: Not Checking User Ownership
**Error**: Users can view/edit other users' data
**Solution**: Always filter by `created_by=request.user`
```python
# Good - Secure
task = get_object_or_404(Task, pk=pk, created_by=request.user)

# Bad - Insecure
task = get_object_or_404(Task, pk=pk)
```

### Mistake 3: Forgetting CSRF Token in Forms
**Error**: `CSRF verification failed`
**Solution**: Always include `{% csrf_token %}` in forms
```django
<form method="POST">
    {% csrf_token %}  <!-- Don't forget this! -->
    <!-- form fields -->
</form>
```

### Mistake 4: Using GET for Data Modification
**Error**: Data can be modified via URL links
**Solution**: Use POST for create/update/delete operations
```python
# Good
if request.method == 'POST':
    # Modify data

# Bad - Never do this
if request.method == 'GET':
    task.delete()  # Dangerous!
```

### Mistake 5: Not Validating Sort Parameters
**Error**: SQL injection vulnerability
**Solution**: Validate user input before using in queries
```python
# Good
allowed_sort = ['title', 'created_at', 'priority']
if sort_column in allowed_sort:
    tasks = tasks.order_by(sort_column)

# Bad
sort_column = request.GET.get('sort')
tasks = tasks.order_by(sort_column)  # Dangerous!
```

---

**Views are ready! Let's build the frontend! 🎨**
