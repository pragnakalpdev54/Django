# Part 6: Class-Based Views - Introduction

## What are Class-Based Views?

Class-Based Views (CBV) are an alternative way to write views in Django using Python classes instead of functions. They provide a more object-oriented approach to handling HTTP requests.

### FBV vs CBV: First Look

#### Example 1: Basic Task List with Filtering

**Function-Based View (15 lines):**
```python
@login_required
def task_list(request):
    tasks = Task.objects.filter(created_by=request.user)
    
    # Apply filters
    status = request.GET.get('status')
    if status:
        tasks = tasks.filter(status=status)
    
    priority = request.GET.get('priority')
    if priority:
        tasks = tasks.filter(priority=priority)
    
    return render(request, 'task_list.html', {'tasks': tasks})
```

**Class-Based View (8 lines with same functionality):**
```python
class TaskListView(LoginRequiredMixin, ListView):
    model = Task
    template_name = 'task_list.html'
    context_object_name = 'tasks'
    
    def get_queryset(self):
        qs = super().get_queryset().filter(created_by=self.request.user)
        return qs.filter(**{k: v for k, v in self.request.GET.items() if v})
```

**What this example shows:**
- FBV requires manual filtering logic for each parameter
- CBV uses `get_queryset()` method for cleaner, reusable filtering
- CBV automatically includes pagination support
- CBV is 47% less code for the same functionality

---

#### Example 2: Adding Pagination

**Function-Based View (25+ lines):**
```python
from django.core.paginator import Paginator, EmptyPage, PageNotAnInteger

@login_required
def task_list(request):
    tasks = Task.objects.filter(created_by=request.user)
    
    # Manual pagination setup
    paginator = Paginator(tasks, 10)
    page = request.GET.get('page')
    
    try:
        tasks = paginator.page(page)
    except PageNotAnInteger:
        tasks = paginator.page(1)
    except EmptyPage:
        tasks = paginator.page(paginator.num_pages)
    
    return render(request, 'task_list.html', {
        'tasks': tasks,
        'paginator': paginator,
        'page_obj': tasks,
    })
```

**Class-Based View (9 lines):**
```python
class TaskListView(LoginRequiredMixin, ListView):
    model = Task
    template_name = 'task_list.html'
    context_object_name = 'tasks'
    paginate_by = 10  # That's it! Pagination done.
    
    def get_queryset(self):
        return Task.objects.filter(created_by=self.request.user)
```

**What this example shows:**
- FBV requires 15+ lines of manual pagination code
- CBV adds pagination with just one attribute: `paginate_by = 10`
- CBV automatically provides `paginator`, `page_obj`, and `is_paginated` context variables
- CBV handles edge cases (invalid page numbers, empty pages) automatically

---

#### Example 3: Reusing Logic Across Views

**Function-Based View - Repeat code in every view:**
```python
@login_required
def task_list(request):
    tasks = Task.objects.filter(created_by=request.user, is_deleted=False)
    return render(request, 'task_list.html', {'tasks': tasks})

@login_required
def task_archive(request):
    tasks = Task.objects.filter(created_by=request.user, is_deleted=False)
    return render(request, 'task_archive.html', {'tasks': tasks})

@login_required
def high_priority_tasks(request):
    tasks = Task.objects.filter(created_by=request.user, is_deleted=False)
    return render(request, 'high_priority.html', {'tasks': tasks})
```

**Class-Based View - Create mixin once, use everywhere:**
```python
class UserTaskMixin:
    def get_queryset(self):
        return super().get_queryset().filter(
            created_by=self.request.user,
            is_deleted=False
        )

class TaskListView(LoginRequiredMixin, UserTaskMixin, ListView):
    model = Task

class TaskArchiveView(LoginRequiredMixin, UserTaskMixin, ListView):
    model = Task

class HighPriorityTaskView(LoginRequiredMixin, UserTaskMixin, ListView):
    model = Task
    def get_queryset(self):
        return super().get_queryset().filter(priority='high')
```

**What this example shows:**
- FBV repeats the same filtering logic in every function
- CBV uses a mixin to define logic once and reuse it
- CBV promotes DRY (Don't Repeat Yourself) principle
- CBV makes it easy to maintain consistent behavior across views

---

#### Example 4: Form Handling

**Function-Based View - Manual GET/POST handling:**
```python
@login_required
def task_create(request):
    if request.method == 'POST':
        form = TaskForm(request.POST)
        if form.is_valid():
            task = form.save(commit=False)
            task.created_by = request.user
            task.save()
            messages.success(request, 'Task created!')
            return redirect('task_list')
    else:
        form = TaskForm()
    
    return render(request, 'task_form.html', {'form': form})
```

**Class-Based View - Automatic handling:**
```python
class TaskCreateView(LoginRequiredMixin, CreateView):
    model = Task
    form_class = TaskForm
    success_url = reverse_lazy('task_list')
    
    def form_valid(self, form):
        form.instance.created_by = self.request.user
        messages.success(self.request, 'Task created!')
        return super().form_valid(form)
```

**What this example shows:**
- FBV requires manual GET/POST method checking
- CBV automatically handles GET (show form) and POST (process form)
- CBV provides built-in form validation and error handling
- CBV reduces boilerplate code significantly for form operations

---

## Why CBV? See It In Action

### Example 1: Adding Pagination with One Line

**FBV - Need to add 10+ lines:**
```python
from django.core.paginator import Paginator

def task_list(request):
    tasks = Task.objects.filter(created_by=request.user)
    
    # Add pagination manually
    paginator = Paginator(tasks, 10)
    page_number = request.GET.get('page')
    page_obj = paginator.get_page(page_number)
    
    return render(request, 'task_list.html', {
        'tasks': page_obj,
        'paginator': paginator,
        'page_obj': page_obj,
    })
```

**CBV - Just add one attribute:**
```python
class TaskListView(LoginRequiredMixin, ListView):
    model = Task
    paginate_by = 10  # That's it! Pagination done.
```

**Why CBV wins here:**
- FBV requires manual paginator setup, error handling, and context management
- CBV handles all pagination logic automatically with one attribute
- CBV provides built-in template variables (`paginator`, `page_obj`, `is_paginated`)
- CBV handles edge cases (invalid page numbers, empty pages) automatically

---

### Example 2: Reusing Logic with Mixins

**FBV - Repeat code in every view:**
```python
@login_required
def task_list(request):
    tasks = Task.objects.filter(created_by=request.user, is_deleted=False)
    return render(request, 'task_list.html', {'tasks': tasks})

@login_required
def task_archive(request):
    tasks = Task.objects.filter(created_by=request.user, is_deleted=False)
    return render(request, 'task_archive.html', {'tasks': tasks})

@login_required
def high_priority_tasks(request):
    tasks = Task.objects.filter(created_by=request.user, is_deleted=False)
    return render(request, 'high_priority.html', {'tasks': tasks})
```

**CBV - Create mixin once, use everywhere:**
```python
class UserTaskMixin:
    def get_queryset(self):
        return super().get_queryset().filter(
            created_by=self.request.user,
            is_deleted=False
        )

class TaskListView(LoginRequiredMixin, UserTaskMixin, ListView):
    model = Task

class TaskArchiveView(LoginRequiredMixin, UserTaskMixin, ListView):
    model = Task

class HighPriorityTaskView(LoginRequiredMixin, UserTaskMixin, ListView):
    model = Task
    
    def get_queryset(self):
        return super().get_queryset().filter(priority='high')
```

**Why CBV wins here:**
- FBV repeats the same filtering logic in every function (violates DRY principle)
- CBV uses a mixin to define logic once and reuse it across multiple views
- CBV makes it easy to maintain consistent behavior - change the mixin, update all views
- CBV promotes code reusability and maintainability

---

### Example 3: Automatic Form Handling

**FBV - Manual GET/POST handling:**
```python
@login_required
def task_create(request):
    if request.method == 'POST':
        form = TaskForm(request.POST)
        if form.is_valid():
            task = form.save(commit=False)
            task.created_by = request.user
            task.save()
            messages.success(request, 'Task created!')
            return redirect('task_list')
    else:
        form = TaskForm()
    
    return render(request, 'task_form.html', {'form': form})
```

**CBV - Automatic handling:**
```python
class TaskCreateView(LoginRequiredMixin, CreateView):
    model = Task
    form_class = TaskForm
    success_url = reverse_lazy('task_list')
    
    def form_valid(self, form):
        form.instance.created_by = self.request.user
        messages.success(self.request, 'Task created!')
        return super().form_valid(form)
```

**What CBV handles automatically:**
- GET request → Display empty form
- POST request → Process form
- Form validation and error display
- Success redirect
- CSRF protection
- Error handling for invalid forms

**Why CBV wins here:**
- FBV requires manual HTTP method checking and form handling
- CBV automatically handles GET/POST logic behind the scenes
- CBV provides built-in form validation and error handling
- CBV reduces boilerplate code by 60-70% for form operations
- CBV follows RESTful conventions automatically

---

## Django's Generic Class-Based Views

Django provides several built-in generic views that handle common web development patterns. These are pre-built classes that save you from writing repetitive code.

### Why Use Generic Views?

**Instead of writing this every time:**
```python
def task_list(request):
    # Get all tasks
    tasks = Task.objects.all()
    
    # Handle pagination
    paginator = Paginator(tasks, 10)
    page = request.GET.get('page')
    
    # Render template
    return render(request, 'task_list.html', {'tasks': tasks})
```

**Just write this:**
```python
class TaskListView(ListView):
    model = Task
    paginate_by = 10
```

**Generic views handle:**
- ✅ Database queries
- ✅ Pagination
- ✅ Template rendering
- ✅ Error handling
- ✅ Common patterns (list, detail, create, update, delete)

### Overview of Generic Views

| View Type | Purpose | Common Use Cases |
|-----------|---------|------------------|
| **ListView** | Display multiple objects | Blog posts, task lists, product catalogs |
| **DetailView** | Display single object | Blog post, task details, user profile |
| **CreateView** | Create new objects | New blog post, add task, user registration |
| **UpdateView** | Update existing objects | Edit blog post, update task, edit profile |
| **DeleteView** | Delete objects | Delete blog post, remove task, delete account |

---

## Next: Complete Reference

For detailed documentation of each generic view including all attributes, methods, and examples, see **Part 7: Class-Based Views - Generic Views & Mixins**.

In the next section, you'll find:
- Complete attribute reference for each view
- Detailed method explanations
- Real-world examples
- Template context variables
- Common customization patterns

---

## Summary

In this introduction, you've learned:
- What Class-Based Views are and their advantages
- How CBV compares to FBV with practical examples
- The basic overview of Django's generic views
- When to use each type of generic view

---

## Common Mistakes to Avoid

### Mistake 1: Wrong Mixin Order
**Error**: Mixins don't work as expected
**Solution**: Order matters! Most specific → Least specific (left to right)
```python
# Good - LoginRequiredMixin first
class TaskListView(LoginRequiredMixin, ListView):
    model = Task

# Bad - ListView first
class TaskListView(ListView, LoginRequiredMixin):  # Won't work!
    model = Task
```

### Mistake 2: Forgetting as_view() in URLs
**Error**: `TypeError: __init__() takes 1 positional argument`
**Solution**: Always call `.as_view()` when using CBV in URLs
```python
# Good
path('tasks/', TaskListView.as_view(), name='task_list'),

# Bad
path('tasks/', TaskListView, name='task_list'),  # Missing .as_view()
```

### Mistake 3: Not Understanding Method Resolution Order (MRO)
**Error**: Methods from wrong mixin are called
**Solution**: Use `super()` to call parent methods properly
```python
def get_queryset(self):
    qs = super().get_queryset()  # Calls parent's method
    return qs.filter(created_by=self.request.user)
```

### Mistake 4: Overriding dispatch() Without Calling super()
**Error**: View doesn't work at all
**Solution**: Always call `super().dispatch()` when overriding
```python
# Good
def dispatch(self, request, *args, **kwargs):
    # Custom logic
    return super().dispatch(request, *args, **kwargs)

# Bad
def dispatch(self, request, *args, **kwargs):
    # Custom logic
    return HttpResponse('Done')  # Skips parent logic!
```

### Mistake 5: Using FBV Decorators on CBV Methods
**Error**: Decorators don't work on CBV methods
**Solution**: Use `@method_decorator` or mixin classes
```python
# Good
from django.utils.decorators import method_decorator

@method_decorator(login_required, name='dispatch')
class TaskListView(ListView):
    model = Task

# Bad
class TaskListView(ListView):
    @login_required  # Won't work!
    def get(self, request):
        pass
```

---

**Ready for more?** Continue to **Part 7** for complete documentation of each generic view with detailed attributes, methods, and real-world examples.
