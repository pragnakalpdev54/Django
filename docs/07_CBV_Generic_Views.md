# Part 7: Class-Based Views - Generic Views & Mixins

## Understanding Mixins - The Power of CBV

Mixins are reusable components that add specific functionality to views. They're the secret weapon that makes CBV powerful.

### What Problem Do Mixins Solve?

**Without Mixins (FBV approach):**
```python
# Repeat this in EVERY view
@login_required
def task_list(request):
    tasks = Task.objects.filter(created_by=request.user)
    return render(request, 'task_list.html', {'tasks': tasks})

@login_required
def task_detail(request, pk):
    task = get_object_or_404(Task, pk=pk, created_by=request.user)
    return render(request, 'task_detail.html', {'task': task})

@login_required
def task_archive(request):
    tasks = Task.objects.filter(created_by=request.user, is_archived=True)
    return render(request, 'task_archive.html', {'tasks': tasks})
```

**With Mixins (CBV approach):**
```python
# Define once
class UserTaskMixin:
    def get_queryset(self):
        return super().get_queryset().filter(created_by=self.request.user)

# Use everywhere - no repetition!
class TaskListView(LoginRequiredMixin, UserTaskMixin, ListView):
    model = Task

class TaskDetailView(LoginRequiredMixin, UserTaskMixin, DetailView):
    model = Task

class TaskArchiveView(LoginRequiredMixin, UserTaskMixin, ListView):
    model = Task
    def get_queryset(self):
        return super().get_queryset().filter(is_archived=True)
```

---

## Built-in Django Mixins - Complete Reference

### LoginRequiredMixin

**Purpose:** Ensure user is authenticated before accessing view.

**Attributes:**
```python
class MyView(LoginRequiredMixin, ListView):
    login_url = '/login/'                    # Where to redirect if not logged in
    redirect_field_name = 'next'             # URL parameter for redirect after login
    permission_denied_message = ''           # Message to display
```

**Example - Before vs After:**
```python
# FBV - Need decorator on every view
@login_required(login_url='login')
def task_list(request):
    pass

@login_required(login_url='login')
def task_detail(request, pk):
    pass

# CBV - Add once to class
class TaskListView(LoginRequiredMixin, ListView):
    model = Task
    login_url = 'login'  # Set once, applies to all HTTP methods
```

### PermissionRequiredMixin

**Purpose:** Check if user has specific permissions.

**Attributes:**
```python
class MyView(PermissionRequiredMixin, CreateView):
    permission_required = 'tasks.add_task'              # Single permission
    permission_required = ['tasks.add_task', 'tasks.change_task']  # Multiple (AND)
    permission_denied_message = 'You cannot add tasks'  # Custom message
    raise_exception = False                             # Raise 403 or redirect
```

**Methods:**
```python
def has_permission(self):
    """Override for custom permission logic"""
    # Check if user owns the object
    obj = self.get_object()
    return obj.created_by == self.request.user

def get_permission_required(self):
    """Dynamic permissions based on conditions"""
    if self.request.user.is_staff:
        return []  # Staff can do anything
    return ['tasks.add_task']
```

**Real Example:**
```python
class TaskUpdateView(PermissionRequiredMixin, UpdateView):
    model = Task
    permission_required = 'tasks.change_task'
    
    def has_permission(self):
        """Only allow editing own tasks"""
        has_perm = super().has_permission()
        if not has_perm:
            return False
        
        # Additional check: user must own the task
        obj = self.get_object()
        return obj.created_by == self.request.user
```

### UserPassesTestMixin

**Purpose:** Custom test function to determine access.

**Methods:**
```python
class MyView(UserPassesTestMixin, UpdateView):
    def test_func(self):
        """Return True to allow access, False to deny"""
        obj = self.get_object()
        return obj.created_by == self.request.user
    
    def handle_no_permission(self):
        """Called when test_func returns False"""
        messages.error(self.request, 'You cannot access this task')
        return redirect('task_list')
```

**Real Examples:**

```python
# Example 1: Check ownership
class TaskUpdateView(UserPassesTestMixin, UpdateView):
    model = Task
    
    def test_func(self):
        task = self.get_object()
        return task.created_by == self.request.user

# Example 2: Check multiple conditions
class TaskDeleteView(UserPassesTestMixin, DeleteView):
    model = Task
    
    def test_func(self):
        task = self.get_object()
        # Can delete if: owner OR staff AND task not completed
        is_owner = task.created_by == self.request.user
        is_staff = self.request.user.is_staff
        not_completed = not task.is_completed
        
        return (is_owner or is_staff) and not_completed

# Example 3: Check subscription status
class PremiumFeatureView(UserPassesTestMixin, TemplateView):
    def test_func(self):
        return self.request.user.profile.is_premium
```

### FormMixin

**Purpose:** Add form handling to views.

**Attributes:**
```python
class MyView(FormMixin, View):
    form_class = MyForm                      # Form to use
    initial = {'key': 'value'}               # Initial form data
    prefix = None                            # Form prefix
    success_url = reverse_lazy('success')    # Where to redirect on success
```

**Methods:**
```python
def get_form_kwargs(self):
    """Pass extra kwargs to form"""
    kwargs = super().get_form_kwargs()
    kwargs['user'] = self.request.user
    return kwargs

def get_initial(self):
    """Set initial form values"""
    initial = super().get_initial()
    initial['priority'] = 'high'
    return initial

def form_valid(self, form):
    """Called when form is valid"""
    # Do something with form
    return super().form_valid(form)

def form_invalid(self, form):
    """Called when form is invalid"""
    messages.error(self.request, 'Form has errors')
    return super().form_invalid(form)
```

---

## Advanced Mixin Patterns

### Pattern 1: Combining Multiple Mixins

**Order matters!** More specific → Less specific (left to right)

```python
class TaskUpdateView(
    LoginRequiredMixin,      # 1. Check authentication first
    PermissionRequiredMixin, # 2. Then check permissions
    UserPassesTestMixin,     # 3. Then custom test
    UpdateView               # 4. Base view last
):
    model = Task
    permission_required = 'tasks.change_task'
    
    def test_func(self):
        # Additional ownership check
        return self.get_object().created_by == self.request.user
```

### Pattern 2: Creating Reusable Mixins

```python
# Mixin 1: Filter by user
class UserFilterMixin:
    """Filter queryset to current user's objects"""
    def get_queryset(self):
        return super().get_queryset().filter(created_by=self.request.user)

# Mixin 2: Exclude deleted
class ExcludeDeletedMixin:
    """Exclude soft-deleted objects"""
    def get_queryset(self):
        return super().get_queryset().filter(is_deleted=False)

# Mixin 3: Add success message
class SuccessMessageMixin:
    """Add success message on form submission"""
    success_message = ""
    
    def form_valid(self, form):
        response = super().form_valid(form)
        if self.success_message:
            messages.success(self.request, self.success_message)
        return response

# Combine them all!
class TaskListView(
    LoginRequiredMixin,
    UserFilterMixin,
    ExcludeDeletedMixin,
    ListView
):
    model = Task

class TaskCreateView(
    LoginRequiredMixin,
    SuccessMessageMixin,
    CreateView
):
    model = Task
    form_class = TaskForm
    success_message = "Task created successfully!"
```

### Pattern 3: Mixin with Properties

```python
class SetUserMixin:
    """Automatically set created_by to current user"""
    def form_valid(self, form):
        form.instance.created_by = self.request.user
        return super().form_valid(form)

class TimestampMixin:
    """Add created_at and updated_at timestamps"""
    def form_valid(self, form):
        if not form.instance.pk:  # New object
            form.instance.created_at = timezone.now()
        form.instance.updated_at = timezone.now()
        return super().form_valid(form)

# Use together
class TaskCreateView(
    LoginRequiredMixin,
    SetUserMixin,
    TimestampMixin,
    CreateView
):
    model = Task
    form_class = TaskForm
```

---

---

## ListView - Complete Reference

### 🎯 When to Use ListView
- Display a list of blog posts
- Show user's tasks
- List products in a category
- Any time you need to show multiple items

### 📋 Attributes Reference

```python
class TaskListView(ListView):
    # Required (choose ONE)
    model = Task                              # Model to query
    queryset = Task.objects.filter(...)      # Or custom queryset
    
    # Template Configuration
    template_name = 'tasks/task_list.html'   # Template file path
    template_name_suffix = '_list'            # Template name suffix
    
    # Context Variables
    context_object_name = 'tasks'             # Variable name in template
    
    # Pagination
    paginate_by = 10                          # Items per page
    paginate_orphans = 3                      # Min items on last page
    paginator_class = Paginator               # Custom paginator class
    
    # Sorting
    ordering = ['-created_at']                # How to sort the list
    
    # HTTP Methods
    http_method_names = ['get', 'post']       # Allowed HTTP methods
```

**Attribute Details:**
- **model** (Model): Required unless using queryset. The model to query objects from.
- **queryset** (QuerySet): Required unless using model. Custom queryset to use instead of model.objects.all().
- **template_name** (String): Template file path. Default: `<app>/<model>_list.html`
- **template_name_suffix** (String): Template name suffix. Default: `_list`
- **context_object_name** (String): Variable name in template. Default: `object_list`
- **paginate_by** (Integer): Number of items per page. None means no pagination.
- **paginate_orphans** (Integer): Minimum items on last page. Default: 0
- **paginator_class** (Class): Custom paginator class. Default: `Paginator`
- **ordering** (String/List): How to sort the list. Default: None
- **http_method_names** (List): Allowed HTTP methods. Default: `['get', 'head', 'options']`

### 🔧 Key Methods

#### `get_queryset(self)`
**Purpose:** Customize what data to display
**Called:** Before the view processes the request

```python
def get_queryset(self):
    """Filter tasks by current user and apply search"""
    qs = super().get_queryset()  # Gets Task.objects.all()
    
    # Filter by user
    qs = qs.filter(created_by=self.request.user)
    
    # Apply search
    search = self.request.GET.get('search')
    if search:
        qs = qs.filter(title__icontains=search)
    
    return qs
```

#### `get_context_data(self, **kwargs)`
**Purpose:** Add extra variables to template
**Called:** After queryset is retrieved

```python
def get_context_data(self, **kwargs):
    """Add statistics and filter values to template"""
    context = super().get_context_data(**kwargs)
    
    # Add statistics
    context['total_tasks'] = self.get_queryset().count()
    context['completed_tasks'] = self.get_queryset().filter(is_completed=True).count()
    
    # Preserve filter values for form
    context['search_query'] = self.request.GET.get('search', '')
    
    return context
```

#### `get_ordering(self)`
**Purpose:** Dynamic sorting based on user input
**Called:** When ordering the queryset

```python
def get_ordering(self):
    """Allow user to choose sorting"""
    order = self.request.GET.get('order', '-created_at')
    return [order]
```

#### `get_paginate_by(self, queryset)`
**Purpose:** Dynamic pagination
**Called:** When setting up pagination

```python
def get_paginate_by(self, queryset):
    """Allow user to choose items per page"""
    per_page = self.request.GET.get('per_page', 10)
    return int(per_page)
```

### 🎨 Template Context Variables

**Available in Template:**
- `object_list` (List): List of objects (default name)
- `tasks` (List): List of objects (if context_object_name = 'tasks')
- `paginator` (Paginator): Paginator object (if paginated)
- `page_obj` (Page): Current page object (if paginated)
- `is_paginated` (Boolean): True if pagination is active

### 📝 Template Example

```django
{# Basic list rendering #}
{% for task in tasks %}
    <div class="task">
        <h3>{{ task.title }}</h3>
        <p>{{ task.description|truncatewords:20 }}</p>
        <small>Created: {{ task.created_at|date:"M d, Y" }}</small>
    </div>
{% empty %}
    <p>No tasks found.</p>
{% endfor %}

{# Pagination controls - Basic #}
{% if is_paginated %}
    <div class="pagination">
        {% if page_obj.has_previous %}
            <a href="?page={{ page_obj.previous_page_number }}">« Previous</a>
        {% endif %}
        
        <span>Page {{ page_obj.number }} of {{ paginator.num_pages }}</span>
        
        {% if page_obj.has_next %}
            <a href="?page={{ page_obj.next_page_number }}">Next »</a>
        {% endif %}
    </div>
{% endif %}
```

### 📄 Complete Pagination Template (Bootstrap 5)

For production-ready pagination with all features:

```django
{# Complete Pagination with Bootstrap 5 #}
{% if is_paginated %}
    <nav aria-label="Page navigation" class="mt-4">
        <ul class="pagination justify-content-center">
            {# First Page #}
            {% if page_obj.has_previous %}
                <li class="page-item">
                    <a class="page-link" href="?page=1" aria-label="First">
                        <span aria-hidden="true">&laquo;&laquo;</span>
                    </a>
                </li>
            {% endif %}
            
            {# Previous Page #}
            {% if page_obj.has_previous %}
                <li class="page-item">
                    <a class="page-link" href="?page={{ page_obj.previous_page_number }}" aria-label="Previous">
                        <span aria-hidden="true">&laquo;</span>
                    </a>
                </li>
            {% else %}
                <li class="page-item disabled">
                    <span class="page-link">&laquo;</span>
                </li>
            {% endif %}
            
            {# Page Numbers (show current +/- 2 pages) #}
            {% for num in paginator.page_range %}
                {% if page_obj.number == num %}
                    <li class="page-item active">
                        <span class="page-link">{{ num }}</span>
                    </li>
                {% elif num > page_obj.number|add:'-3' and num < page_obj.number|add:'3' %}
                    <li class="page-item">
                        <a class="page-link" href="?page={{ num }}">{{ num }}</a>
                    </li>
                {% endif %}
            {% endfor %}
            
            {# Next Page #}
            {% if page_obj.has_next %}
                <li class="page-item">
                    <a class="page-link" href="?page={{ page_obj.next_page_number }}" aria-label="Next">
                        <span aria-hidden="true">&raquo;</span>
                    </a>
                </li>
            {% else %}
                <li class="page-item disabled">
                    <span class="page-link">&raquo;</span>
                </li>
            {% endif %}
            
            {# Last Page #}
            {% if page_obj.has_next %}
                <li class="page-item">
                    <a class="page-link" href="?page={{ paginator.num_pages }}" aria-label="Last">
                        <span aria-hidden="true">&raquo;&raquo;</span>
                    </a>
                </li>
            {% endif %}
        </ul>
        
        {# Page Info #}
        <p class="text-center text-muted">
            Page {{ page_obj.number }} of {{ paginator.num_pages }} 
            ({{ paginator.count }} total items)
        </p>
    </nav>
{% endif %}
```

**Pagination Variables:**
- `is_paginated` - Boolean, True if pagination is active
- `page_obj.number` - Current page number
- `page_obj.has_previous()` - Boolean, True if previous page exists
- `page_obj.has_next()` - Boolean, True if next page exists
- `page_obj.previous_page_number` - Previous page number
- `page_obj.next_page_number` - Next page number
- `paginator.num_pages` - Total number of pages
- `paginator.count` - Total number of objects
- `paginator.page_range` - Range of all page numbers
```

### 🚀 Real-World Example

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.db.models import Q
from django.views.generic import ListView
from .models import Task

class TaskListView(LoginRequiredMixin, ListView):
    """Advanced task list with filtering, search, and pagination"""
    model = Task
    template_name = 'tasks/task_list.html'
    context_object_name = 'tasks'
    paginate_by = 20
    login_url = 'login'
    
    def get_queryset(self):
        """Apply filters and search"""
        qs = Task.objects.filter(created_by=self.request.user, is_deleted=False)
        
        # Status filter
        status = self.request.GET.get('status')
        if status and status != 'all':
            qs = qs.filter(status=status)
        
        # Priority filter
        priority = self.request.GET.get('priority')
        if priority and priority != 'all':
            qs = qs.filter(priority=priority)
        
        # Search across title and description
        search = self.request.GET.get('q')
        if search:
            qs = qs.filter(
                Q(title__icontains=search) | 
                Q(description__icontains=search)
            )
        
        # Apply ordering
        order = self.request.GET.get('order', '-created_at')
        qs = qs.order_by(order)
        
        return qs
    
    def get_context_data(self, **kwargs):
        """Add filter options and statistics"""
        context = super().get_context_data(**kwargs)
        
        # Current filter values
        context['current_status'] = self.request.GET.get('status', 'all')
        context['current_priority'] = self.request.GET.get('priority', 'all')
        context['current_order'] = self.request.GET.get('order', '-created_at')
        context['search_query'] = self.request.GET.get('q', '')
        
        # Statistics
        all_tasks = Task.objects.filter(created_by=self.request.user, is_deleted=False)
        context['stats'] = {
            'total': all_tasks.count(),
            'pending': all_tasks.filter(status='pending').count(),
            'in_progress': all_tasks.filter(status='in_progress').count(),
            'completed': all_tasks.filter(status='completed').count(),
        }
        
        return context
```

---

## DetailView - Complete Reference

### 🎯 When to Use DetailView
- Show a single blog post
- Display task details
- View user profile
- Any time you need to show one specific item

### 📋 Attributes Reference

```python
class TaskDetailView(DetailView):
    # Required (choose ONE)
    model = Task                              # Model to query
    queryset = Task.objects.all()            # Or custom queryset
    
    # Template Configuration
    template_name = 'tasks/task_detail.html' # Template file path
    template_name_suffix = '_detail'          # Template name suffix
    
    # Context Variables
    context_object_name = 'task'              # Variable name in template
    
    # URL Configuration
    pk_url_kwarg = 'pk'                       # URL parameter name for primary key
    slug_url_kwarg = 'slug'                   # URL parameter name for slug
    slug_field = 'slug'                       # Model field to match slug against
    
    # Query Options
    query_pk_and_slug = False                 # Require both pk and slug
```

**Attribute Details:**
- **model** (Model): Required unless using queryset. The model to query objects from.
- **queryset** (QuerySet): Required unless using model. Custom queryset to use instead of model.objects.all().
- **template_name** (String): Template file path. Default: `<app>/<model>_detail.html`
- **template_name_suffix** (String): Template name suffix. Default: `_detail`
- **context_object_name** (String): Variable name in template. Default: `object`
- **pk_url_kwarg** (String): URL parameter name for primary key. Default: `pk`
- **slug_url_kwarg** (String): URL parameter name for slug. Default: `slug`
- **slug_field** (String): Model field to match slug against. Default: `slug`
- **query_pk_and_slug** (Boolean): Require both pk and slug. Default: `False`

### 🔧 Key Methods

#### `get_object(self, queryset=None)`
**Purpose:** Customize how the object is retrieved
**Called:** When the view needs the object

```python
def get_object(self, queryset=None):
    """Add view count and check permissions"""
    obj = super().get_object(queryset)
    
    # Check if user owns the object
    if obj.created_by != self.request.user:
        raise Http404("Task not found")
    
    # Add view count
    obj.view_count += 1
    obj.save(update_fields=['view_count'])
    
    return obj
```

#### `get_queryset(self)`
**Purpose:** Filter what objects can be viewed
**Called:** Before retrieving the object

```python
def get_queryset(self):
    """Only show user's own non-deleted tasks"""
    return Task.objects.filter(
        created_by=self.request.user,
        is_deleted=False
    ).select_related('created_by')  # Optimize query
```

#### `get_context_data(self, **kwargs)`
**Purpose:** Add related data to template
**Called:** After object is retrieved

```python
def get_context_data(self, **kwargs):
    """Add related tasks and user permissions"""
    context = super().get_context_data(**kwargs)
    
    # Add related tasks with same priority
    context['similar_tasks'] = Task.objects.filter(
        created_by=self.request.user,
        priority=self.object.priority,
        is_deleted=False
    ).exclude(pk=self.object.pk)[:5]
    
    # Check if user can edit
    context['can_edit'] = self.object.created_by == self.request.user
    context['can_delete'] = not self.object.is_completed
    
    return context
```

### 🎨 Template Context Variables

**Available in Template:**
- `object` (Model): The retrieved object (default name)
- `task` (Model): The retrieved object (if context_object_name = 'task')

### 📝 Template Example

```django
{# Basic object display #}
<div class="task-detail">
    <h1>{{ task.title }}</h1>
    <p class="description">{{ task.description }}</p>
    
    <div class="meta">
        <p><strong>Priority:</strong> {{ task.get_priority_display }}</p>
        <p><strong>Status:</strong> {{ task.get_status_display }}</p>
        <p><strong>Created:</strong> {{ task.created_at|date:"F d, Y" }}</p>
        {% if task.due_date %}
            <p><strong>Due:</strong> {{ task.due_date|date:"F d, Y" }}</p>
        {% endif %}
    </div>
</div>

{# Related objects from context #}
{% if similar_tasks %}
    <div class="related-tasks">
        <h3>Similar Tasks</h3>
        {% for task in similar_tasks %}
            <a href="{% url 'task_detail' task.pk %}">{{ task.title }}</a>
        {% endfor %}
    </div>
{% endif %}

{# Conditional actions based on context #}
{% if can_edit %}
    <a href="{% url 'task_update' task.pk %}" class="btn btn-primary">Edit Task</a>
{% endif %}
{% if can_delete %}
    <a href="{% url 'task_delete' task.pk %}" class="btn btn-danger">Delete Task</a>
{% endif %}
```

### 🚀 Real-World Example

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.http import Http404
from django.views.generic import DetailView
from .models import Task

class TaskDetailView(LoginRequiredMixin, DetailView):
    """Advanced task detail with tracking and related content"""
    model = Task
    template_name = 'tasks/task_detail.html'
    context_object_name = 'task'
    
    def get_queryset(self):
        """Only show user's own non-deleted tasks"""
        return Task.objects.filter(
            created_by=self.request.user,
            is_deleted=False
        ).select_related('created_by').prefetch_related('subtasks')
    
    def get_object(self, queryset=None):
        """Add tracking and validation"""
        obj = super().get_object(queryset)
        
        # Track last viewed time
        self.request.session[f'task_{obj.pk}_viewed'] = timezone.now().isoformat()
        
        return obj
    
    def get_context_data(self, **kwargs):
        """Add comprehensive context for detail view"""
        context = super().get_context_data(**kwargs)
        task = self.object
        
        # Similar tasks
        context['similar_tasks'] = Task.objects.filter(
            created_by=self.request.user,
            priority=task.priority,
            is_deleted=False
        ).exclude(pk=task.pk)[:5]
        
        # Task hierarchy
        context['subtasks'] = task.subtasks.filter(is_deleted=False)
        if task.parent_task:
            context['parent_task'] = task.parent_task
        
        # Permissions
        context['can_edit'] = (
            task.created_by == self.request.user and 
            not task.is_completed
        )
        context['can_delete'] = (
            task.created_by == self.request.user and
            not task.is_completed
        )
        
        return context
```

---

## CreateView - Complete Reference

### 🎯 When to Use CreateView
- Create new blog posts
- Add new tasks
- User registration forms
- Any form that creates a new database record

### 📋 Attributes Reference

```python
class TaskCreateView(CreateView):
    # Required (choose ONE or MORE)
    model = Task                              # Model to create
    form_class = TaskForm                     # Or custom form class
    fields = ['title', 'description']         # Or specify fields directly
    
    # Template Configuration
    template_name = 'tasks/task_form.html'   # Template file path
    template_name_suffix = '_form'            # Template name suffix
    
    # Success Configuration
    success_url = reverse_lazy('task_list')   # Where to redirect after success
    
    # Form Configuration
    initial = {'priority': 'medium'}          # Initial form values
    prefix = None                             # Form prefix for multiple forms
```

**Attribute Details:**
- **model** (Model): Required unless using form_class/fields. The model to create objects from.
- **form_class** (FormClass): Required unless using model/fields. Custom form class to use.
- **fields** (List/Tuple): Required unless using model/form_class. Fields to include in form.
- **template_name** (String): Template file path. Default: `<app>/<model>_form.html`
- **template_name_suffix** (String): Template name suffix. Default: `_form`
- **success_url** (String): Where to redirect after success. Default: None
- **initial** (Dict): Initial form values. Default: {}
- **prefix** (String): Form prefix for multiple forms. Default: None

### 🔧 Key Methods

#### `form_valid(self, form)`
**Purpose:** Called when form is valid (before saving)
**Called:** After form validation passes

```python
def form_valid(self, form):
    """Set created_by and add success message"""
    # Set fields before saving
    form.instance.created_by = self.request.user
    form.instance.ip_address = self.request.META.get('REMOTE_ADDR')
    
    # Add success message
    messages.success(self.request, f'Task "{form.instance.title}" created!')
    
    # Save and redirect
    return super().form_valid(form)
```

#### `form_invalid(self, form)`
**Purpose:** Called when form has errors
**Called:** After form validation fails

```python
def form_invalid(self, form):
    """Add error message and return response"""
    messages.error(self.request, 'Please correct the errors below.')
    return super().form_invalid(form)
```

#### `get_initial(self)`
**Purpose:** Set initial form values dynamically
**Called:** When creating the form instance

```python
def get_initial(self):
    """Set initial values from URL parameters"""
    initial = super().get_initial()
    initial['priority'] = self.request.GET.get('priority', 'medium')
    initial['due_date'] = timezone.now().date() + timedelta(days=7)
    return initial
```

#### `get_success_url(self)`
**Purpose:** Where to redirect after successful creation
**Called:** After successful form submission

```python
def get_success_url(self):
    """Go to detail page of created object"""
    return reverse('task_detail', kwargs={'pk': self.object.pk})
```

### 🎨 What CreateView Handles Automatically

- ✅ **GET request**: Shows empty form
- ✅ **POST request**: Processes form data
- ✅ **Form validation**: Checks all fields
- ✅ **Error display**: Shows errors in template
- ✅ **Object creation**: Saves to database
- ✅ **Success redirect**: Redirects after save
- ✅ **CSRF protection**: Prevents cross-site attacks

### 📝 Template Example

```django
{# Form for creating new object #}
<form method="post">
    {% csrf_token %}
    
    {{ form.as_p }}
    
    {# Display form errors #}
    {% if form.errors %}
        <div class="errorlist">
            {% for field, errors in form.errors.items %}
                <div class="field-error">
                    <strong>{{ field }}:</strong>
                    {% for error in errors %}
                        {{ error }}
                    {% endfor %}
                </div>
            {% endfor %}
        </div>
    {% endif %}
    
    <button type="submit" class="btn btn-primary">Create Task</button>
</form>
```

### 🚀 Real-World Example

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.contrib import messages
from django.urls import reverse_lazy
from django.views.generic import CreateView
from .forms import TaskForm
from .models import Task

class TaskCreateView(LoginRequiredMixin, CreateView):
    """Advanced task creation with validation and notifications"""
    model = Task
    form_class = TaskForm
    template_name = 'tasks/task_form.html'
    success_url = reverse_lazy('task_list')
    
    def form_valid(self, form):
        """Set creator and apply business logic"""
        # Set the creator
        form.instance.created_by = self.request.user
        
        # Apply business rules
        if form.instance.priority == 'high':
            # High priority tasks due tomorrow
            form.instance.due_date = timezone.now().date() + timedelta(days=1)
            messages.warning(
                self.request,
                'High priority task due date set to tomorrow'
            )
        
        # Save and get response
        response = super().form_valid(form)
        
        # Add success message
        messages.success(
            self.request,
            f'Task "{self.object.title}" created successfully!'
        )
        
        return response
    
    def get_success_url(self):
        """Dynamic success URL based on user action"""
        # Check if user wants to create another
        if 'save_and_add' in self.request.POST:
            return reverse('task_create')
        return reverse('task_detail', kwargs={'pk': self.object.pk})
```

---

## UpdateView - Complete Reference

### 🎯 When to Use UpdateView
- Edit blog posts
- Update task information
- User profile editing
- Any form that modifies an existing record

### 📋 Attributes Reference

```python
class TaskUpdateView(UpdateView):
    # Required (choose ONE or MORE)
    model = Task                              # Model to update
    form_class = TaskForm                     # Or custom form class
    fields = ['title', 'description']         # Or specify fields directly
    
    # Template Configuration
    template_name = 'tasks/task_form.html'   # Template file path
    template_name_suffix = '_form'            # Template name suffix
    
    # Success Configuration
    success_url = reverse_lazy('task_list')   # Where to redirect after success
    
    # URL Configuration
    pk_url_kwarg = 'pk'                       # URL parameter name for primary key
    slug_url_kwarg = 'slug'                   # URL parameter name for slug
```

**Attribute Details:**
- **model** (Model): Required unless using form_class/fields. The model to update objects from.
- **form_class** (FormClass): Required unless using model/fields. Custom form class to use.
- **fields** (List/Tuple): Required unless using model/form_class. Fields to include in form.
- **template_name** (String): Template file path. Default: `<app>/<model>_form.html`
- **template_name_suffix** (String): Template name suffix. Default: `_form`
- **success_url** (String): Where to redirect after success. Default: None
- **pk_url_kwarg** (String): URL parameter name for primary key. Default: `pk`
- **slug_url_kwarg** (String): URL parameter name for slug. Default: `slug`

### 🔧 Key Methods

#### `get_queryset(self)`
**Purpose:** Filter objects that can be updated
**Called:** Before retrieving the object

```python
def get_queryset(self):
    """Only allow updating user's own tasks"""
    return Task.objects.filter(
        created_by=self.request.user,
        is_deleted=False
    )
```

#### `form_valid(self, form)`
**Purpose:** Called when form is valid (before saving)
**Called:** After form validation passes

```python
def form_valid(self, form):
    """Track changes and add timestamps"""
    # Track what changed
    if 'status' in form.changed_data:
        old_status = Task.objects.get(pk=self.object.pk).status
        new_status = form.cleaned_data['status']
        messages.info(
            self.request,
            f'Status changed from {old_status} to {new_status}'
        )
    
    # Update modified timestamp
    form.instance.updated_at = timezone.now()
    form.instance.updated_by = self.request.user
    
    messages.success(self.request, 'Task updated!')
    return super().form_valid(form)
```

### 🎨 What UpdateView Handles Automatically

- ✅ **GET request**: Shows form with existing data
- ✅ **POST request**: Processes form data
- ✅ **Object retrieval**: Gets object by ID from URL
- ✅ **Form population**: Fills form with current values
- ✅ **Form validation**: Checks all fields
- ✅ **Object update**: Saves changes to database
- ✅ **Success redirect**: Redirects after save

### 📝 Template Example

```django
{# Form for updating existing object #}
<form method="post">
    {% csrf_token %}
    
    {{ form.as_p }}
    
    {# Show what changed (from context) #}
    {% if changed_fields %}
        <div class="changes-preview">
            <h4>Changes to be made:</h4>
            {% for field in changed_fields %}
                <div class="change-item">
                    {{ field|title }} will be updated
                </div>
            {% endfor %}
        </div>
    {% endif %}
    
    <button type="submit" class="btn btn-primary">Save Changes</button>
    <a href="{% url 'task_detail' object.pk %}" class="btn btn-secondary">Cancel</a>
</form>
```

### 🚀 Real-World Example

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.contrib import messages
from django.urls import reverse_lazy
from django.views.generic import UpdateView
from .forms import TaskForm
from .models import Task

class TaskUpdateView(LoginRequiredMixin, UpdateView):
    """Advanced task update with change tracking"""
    model = Task
    form_class = TaskForm
    template_name = 'tasks/task_form.html'
    success_url = reverse_lazy('task_list')
    
    def get_queryset(self):
        """Ensure user can only update their own tasks"""
        return Task.objects.filter(
            created_by=self.request.user,
            is_deleted=False
        )
    
    def form_valid(self, form):
        """Track important changes and apply business logic"""
        # Store original values before update
        original = Task.objects.get(pk=self.object.pk)
        
        # Check for important changes
        if form.cleaned_data['priority'] != original.priority:
            if form.cleaned_data['priority'] == 'high':
                # Send notification for high priority
                messages.warning(
                    self.request,
                    'Task marked as high priority - due date adjusted'
                )
                form.instance.due_date = timezone.now().date() + timedelta(days=1)
        
        # Mark as modified
        form.instance.updated_at = timezone.now()
        form.instance.updated_by = self.request.user
        
        response = super().form_valid(form)
        
        messages.success(self.request, f'Task "{self.object.title}" updated!')
        return response
    
    def get_success_url(self):
        """Return to detail page after update"""
        return reverse('task_detail', kwargs={'pk': self.object.pk})
```

---

## DeleteView - Complete Reference

### 🎯 When to Use DeleteView
- Delete blog posts
- Remove tasks
- Delete user accounts
- Any action that removes a database record

### 📋 Attributes Reference

```python
class TaskDeleteView(DeleteView):
    # Required
    model = Task                              # Model to delete
    
    # Template Configuration
    template_name = 'tasks/task_confirm_delete.html'  # Template file path
    template_name_suffix = '_confirm_delete'  # Template name suffix
    
    # Success Configuration
    success_url = reverse_lazy('task_list')   # Where to redirect after deletion
    
    # URL Configuration
    pk_url_kwarg = 'pk'                       # URL parameter name for primary key
    slug_url_kwarg = 'slug'                   # URL parameter name for slug
```

**Attribute Details:**
- **model** (Model): Required. The model to delete objects from.
- **template_name** (String): Template file path. Default: `<app>/<model>_confirm_delete.html`
- **template_name_suffix** (String): Template name suffix. Default: `_confirm_delete`
- **success_url** (String): Required. Where to redirect after deletion.
- **pk_url_kwarg** (String): URL parameter name for primary key. Default: `pk`
- **slug_url_kwarg** (String): URL parameter name for slug. Default: `slug`

### 🔧 Key Methods

#### `get_queryset(self)`
**Purpose:** Filter objects that can be deleted
**Called:** Before retrieving the object

```python
def get_queryset(self):
    """Only allow deleting user's own tasks"""
    return Task.objects.filter(
        created_by=self.request.user,
        is_deleted=False
    )
```

#### `delete(self, request, *args, **kwargs)`
**Purpose:** Handle the deletion process
**Called:** When POST request is confirmed

```python
def delete(self, request, *args, **kwargs):
    """Soft delete with logging"""
    self.object = self.get_object()
    task_title = self.object.title
    
    # Soft delete instead of hard delete
    self.object.is_deleted = True
    self.object.deleted_at = timezone.now()
    self.object.deleted_by = request.user
    self.object.save()
    
    messages.success(request, f'Task "{task_title}" moved to trash.')
    return HttpResponseRedirect(self.get_success_url())
```

### 🎨 What DeleteView Handles Automatically

- ✅ **GET request**: Shows confirmation page
- ✅ **POST request**: Performs deletion
- ✅ **Object retrieval**: Gets object by ID from URL
- ✅ **Confirmation**: Asks user before deleting
- ✅ **Deletion**: Removes from database
- ✅ **Success redirect**: Redirects after delete

### 📝 Template Example

```django
{# Confirmation page for deletion #}
<div class="delete-confirmation">
    <h2>Delete Task</h2>
    
    <p>Are you sure you want to delete "<strong>{{ task.title }}</strong>"?</p>
    
    <form method="post">
        {% csrf_token %}
        <button type="submit" class="btn btn-danger">Yes, Delete</button>
        <a href="{% url 'task_detail' task.pk %}" class="btn btn-secondary">Cancel</a>
    </form>
</div>
```

### 🚀 Real-World Example

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.contrib import messages
from django.http import HttpResponseRedirect
from django.urls import reverse_lazy
from django.views.generic import DeleteView
from .models import Task

class TaskDeleteView(LoginRequiredMixin, DeleteView):
    """Soft delete with logging and dependency checking"""
    model = Task
    template_name = 'tasks/task_confirm_delete.html'
    success_url = reverse_lazy('task_list')
    
    def get_queryset(self):
        """Only allow deletion of user's own tasks"""
        return Task.objects.filter(
            created_by=self.request.user,
            is_deleted=False
        )
    
    def delete(self, request, *args, **kwargs):
        """Soft delete with comprehensive logging"""
        self.object = self.get_object()
        task_title = self.object.title
        
        # Soft delete
        self.object.is_deleted = True
        self.object.deleted_at = timezone.now()
        self.object.deleted_by = request.user
        self.object.save(update_fields=['is_deleted', 'deleted_at', 'deleted_by'])
        
        messages.success(
            request,
            f'Task "{task_title}" moved to trash. You can restore it within 30 days.'
        )
        
        return HttpResponseRedirect(self.get_success_url())
```

---

## TemplateView - Complete Reference

### 🎯 When to Use TemplateView
- Static pages (About, Contact, FAQ)
- Dashboard pages with custom context
- Pages that don't need database operations
- Landing pages

### 📋 Attributes Reference

```python
class DashboardView(TemplateView):
    # Required
    template_name = 'dashboard.html'           # Template file path
    
    # Optional
    template_engine = None                     # Template engine to use
    response_class = TemplateResponse          # Custom response class
```

**Attribute Details:**
- **template_name** (String): Required. Template file path.
- **template_engine** (String): Template engine to use. Default: None
- **response_class** (Class): Custom response class. Default: `TemplateResponse`

### 🔧 Key Methods

#### `get_context_data(self, **kwargs)`
**Purpose:** Add context variables to template
**Called:** When rendering the template

```python
def get_context_data(self, **kwargs):
    """Add user statistics to dashboard"""
    context = super().get_context_data(**kwargs)
    
    # Add user data
    context['user_tasks'] = Task.objects.filter(
        created_by=self.request.user
    ).count()
    
    return context
```

### 📝 Template Example

```django
{# Static About page #}
<div class="about-page">
    <h1>About Our Task Manager</h1>
    <p>Learn more about our amazing application...</p>
    
    {% if user_tasks %}
        <p>You have {{ user_tasks }} tasks created!</p>
    {% endif %}
</div>
```

### 🚀 Real-World Example

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import TemplateView
from .models import Task

class DashboardView(LoginRequiredMixin, TemplateView):
    """User dashboard with statistics"""
    template_name = 'dashboard.html'
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        user = self.request.user
        
        # Statistics
        context['stats'] = {
            'total_tasks': Task.objects.filter(created_by=user).count(),
            'completed': Task.objects.filter(
                created_by=user, 
                is_completed=True
            ).count(),
            'pending': Task.objects.filter(
                created_by=user, 
                is_completed=False
            ).count(),
        }
        
        return context
```

---

## RedirectView - Complete Reference

### 🎯 When to Use RedirectView
- Simple URL redirects
- Legacy URL mapping
- Temporary redirects
- External link redirects

### 📋 Attributes Reference

```python
class LegacyTaskRedirectView(RedirectView):
    # Required
    url = '/new-url/'                          # URL to redirect to
    
    # Optional
    permanent = False                          # Use 301 (permanent) or 302 (temporary)
    query_string = True                        # Pass query string to redirect URL
```

**Attribute Details:**
- **url** (String): Required. URL to redirect to.
- **permanent** (Boolean): Use 301 (permanent) or 302 (temporary). Default: `False`
- **query_string** (Boolean): Pass query string to redirect URL. Default: `False`

### 🔧 Key Methods

#### `get_redirect_url(self, *args, **kwargs)`
**Purpose:** Dynamic URL generation
**Called:** When determining where to redirect

```python
def get_redirect_url(self, *args, **kwargs):
    """Redirect based on user type"""
    if self.request.user.is_staff:
        return '/admin/'
    return '/dashboard/'
```

### 🚀 Real-World Example

```python
from django.views.generic import RedirectView
from django.urls import reverse

class LegacyTaskRedirectView(RedirectView):
    """Redirect old task URLs to new ones"""
    permanent = True  # 301 redirect for SEO
    query_string = True
    
    def get_redirect_url(self, *args, **kwargs):
        """Map old task IDs to new URLs"""
        old_id = kwargs.get('old_id')
        if old_id:
            new_id = self.map_old_id_to_new(old_id)
            return reverse('task-detail', kwargs={'pk': new_id})
        return reverse('task-list')
    
    def map_old_id_to_new(self, old_id):
        """Example mapping function"""
        return old_id  # For this example, keep same ID

class ExternalDocsRedirectView(RedirectView):
    """Redirect to external documentation"""
    url = 'https://docs.example.com'
    permanent = False
```

---

## FormView - Complete Reference

### 🎯 When to Use FormView
- Contact forms
- Search forms
- Forms not tied to models
- Multi-step forms
- Custom form processing

### 📋 Attributes Reference

```python
class ContactView(FormView):
    # Required
    form_class = ContactForm                   # Form class to use
    
    # Optional
    template_name = 'contact.html'             # Template file path
    success_url = reverse_lazy('contact-success')  # Where to redirect after success
    initial = {}                               # Initial form values
```

**Attribute Details:**
- **form_class** (FormClass): Required. Form class to use.
- **template_name** (String): Template file path. Default: `<app>/<form_name>.html`
- **success_url** (String): Where to redirect after success. Default: None
- **initial** (Dict): Initial form values. Default: {}

### 🔧 Key Methods

#### `form_valid(self, form)`
**Purpose:** Process valid form data
**Called:** After form validation passes

```python
def form_valid(self, form):
    """Send email and redirect"""
    # Send contact email
    send_mail(
        subject=form.cleaned_data['subject'],
        message=form.cleaned_data['message'],
        from_email=form.cleaned_data['email'],
        recipient_list=['contact@example.com'],
    )
    
    messages.success(self.request, 'Message sent successfully!')
    return super().form_valid(form)
```

### 🚀 Real-World Example

```python
from django.views.generic import FormView
from django.urls import reverse_lazy
from django.core.mail import send_mail
from django.contrib import messages
from .forms import ContactForm

class ContactView(FormView):
    """Contact form with email sending"""
    form_class = ContactForm
    template_name = 'contact.html'
    success_url = reverse_lazy('contact-success')
    
    def form_valid(self, form):
        """Process contact form"""
        cleaned_data = form.cleaned_data
        
        # Send email
        send_mail(
            subject=f"Contact: {cleaned_data['subject']}",
            message=f"From: {cleaned_data['name']} ({cleaned_data['email']})\n\n"
                   f"{cleaned_data['message']}",
            from_email=cleaned_data['email'],
            recipient_list=['contact@example.com'],
            fail_silently=False,
        )
        
        messages.success(
            self.request,
            'Your message has been sent. We\'ll respond within 24 hours.'
        )
        
        return super().form_valid(form)
```

---

## 📚 Quick Reference Summary

### View Selection Guide

**When to Use Each View:**
- **List items** → `ListView` (key attribute: `model`)
- **Show one item** → `DetailView` (key attribute: `model`)
- **Create new item** → `CreateView` (key attribute: `model` or `form_class`)
- **Edit existing item** → `UpdateView` (key attribute: `model` or `form_class`)
- **Delete item** → `DeleteView` (key attribute: `model`)
- **Static page** → `TemplateView` (key attribute: `template_name`)
- **Simple redirect** → `RedirectView` (key attribute: `url`)
- **Custom form** → `FormView` (key attribute: `form_class`)

### Common Method Override Points

**Methods and When to Override:**
- `get_queryset()` → Filter data (user permissions, soft delete)
- `get_context_data()` → Add template variables (statistics, related data)
- `form_valid()` → Process valid forms (set fields, send notifications)
- `get_success_url()` → Custom redirect (go to created/updated object)
- `get_object()` → Customize object retrieval (add tracking, permissions)

### Essential Mixins Order

```python
# Correct order (most specific → least specific)
class MyView(
    LoginRequiredMixin,      # 1. Authentication
    PermissionRequiredMixin, # 2. Permissions  
    UserPassesTestMixin,     # 3. Custom tests
    ListView                # 4. Base view
):
    pass
```

### Template Context Variables

**Default Variables by View Type:**
- **ListView** → `object_list` (or `context_object_name`)
- **DetailView** → `object` (or `context_object_name`)
- **FormViews** → `form`
- **All Views** → `request`, `view`

---

## 🎯 Best Practices

### ✅ Do's
- **Use mixins** for reusable functionality
- **Override specific methods** instead of the entire view
- **Filter in `get_queryset()`** for security
- **Add context in `get_context_data()`** for templates
- **Use `reverse_lazy()`** for class-level URLs

### ❌ Don'ts
- **Don't override `dispatch()`** unless absolutely necessary
- **Don't hardcode URLs** in views
- **Don't forget to call `super()`** in method overrides
- **Don't mix GET/POST logic** in the same method
- **Don't skip permission checks**

---

## 🚀 Next Steps

Now that you understand Django's generic views:

1. **Practice** by building a simple CRUD application
2. **Explore** more advanced mixins and patterns
3. **Learn** about custom generic views
4. **Study** the Django source code for deeper understanding
5. **Build** your own reusable mixins

**Remember:** Generic views are tools - use them when they make your code cleaner, not just because they exist!

---

## 🌐 CBV with HTML + AJAX (Modern API Approach)

### 🎯 Why Use HTML + AJAX Instead of Jinja?

**Traditional Jinja Approach:**
- Server renders complete HTML
- Full page reloads
- Mixed logic in templates
- Harder to build SPAs

**Modern HTML + AJAX Approach:**
- API endpoints return JSON
- Frontend handles rendering
- Better separation of concerns
- Supports SPAs and mobile apps

### 📋 Creating API Endpoints with CBV

#### Method 1: JSONResponseMixin

```python
from django.http import JsonResponse
from django.views.generic import View
import json

class JSONResponseMixin:
    """Mixin to add JSON response capability"""
    def render_to_json_response(self, context, **response_kwargs):
        """Convert context to JSON response"""
        return JsonResponse(context, **response_kwargs)

class TaskListAPIView(JSONResponseMixin, LoginRequiredMixin, ListView):
    """API endpoint for task list"""
    model = Task
    context_object_name = 'tasks'
    
    def get_queryset(self):
        return Task.objects.filter(
            created_by=self.request.user,
            is_deleted=False
        ).values('id', 'title', 'status', 'priority', 'created_at', 'due_date')
    
    def render_to_response(self, context, **response_kwargs):
        """Override to return JSON instead of template"""
        # Convert queryset to list of dicts
        tasks_data = list(context['tasks'])
        
        # Format dates
        for task in tasks_data:
            task['created_at'] = task['created_at'].isoformat()
            if task['due_date']:
                task['due_date'] = task['due_date'].isoformat()
        
        return self.render_to_json_response({
            'tasks': tasks_data,
            'total': len(tasks_data),
            'status': 'success'
        })
```

#### Method 2: Check Accept Header

```python
from django.http import JsonResponse
from django.views.generic import DetailView

class TaskDetailAPIView(LoginRequiredMixin, DetailView):
    """API endpoint that returns JSON for AJAX requests"""
    model = Task
    
    def get_queryset(self):
        return Task.objects.filter(
            created_by=self.request.user,
            is_deleted=False
        )
    
    def render_to_response(self, context, **response_kwargs):
        """Return JSON for AJAX, HTML for normal requests"""
        if self.request.headers.get('Accept') == 'application/json':
            task = context['task']
            return JsonResponse({
                'id': task.id,
                'title': task.title,
                'description': task.description,
                'status': task.status,
                'priority': task.priority,
                'created_at': task.created_at.isoformat(),
                'due_date': task.due_date.isoformat() if task.due_date else None,
                'is_completed': task.is_completed,
            })
        
        # Fall back to normal template rendering
        return super().render_to_response(context, **response_kwargs)
```

### 🔧 Form Handling with AJAX

#### CreateView with AJAX Support

```python
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator

@method_decorator(csrf_exempt, name='dispatch')
class TaskCreateAPIView(LoginRequiredMixin, CreateView):
    """API endpoint for creating tasks via AJAX"""
    model = Task
    fields = ['title', 'description', 'priority', 'due_date']
    
    def form_valid(self, form):
        """Handle AJAX form submission"""
        self.object = form.save()
        
        # Add created_by
        self.object.created_by = self.request.user
        self.object.save()
        
        # Return JSON response
        return JsonResponse({
            'status': 'success',
            'message': 'Task created successfully',
            'task': {
                'id': self.object.id,
                'title': self.object.title,
                'status': self.object.status,
                'priority': self.object.priority,
                'created_at': self.object.created_at.isoformat(),
            }
        })
    
    def form_invalid(self, form):
        """Handle AJAX form errors"""
        return JsonResponse({
            'status': 'error',
            'message': 'Please correct the errors below',
            'errors': form.errors
        }, status=400)
    
    def get(self, request, *args, **kwargs):
        """Handle GET request - return empty form"""
        return JsonResponse({
            'status': 'info',
            'message': 'Send POST request to create task'
        })
```

#### UpdateView with AJAX Support

```python
class TaskUpdateAPIView(LoginRequiredMixin, UpdateView):
    """API endpoint for updating tasks via AJAX"""
    model = Task
    fields = ['title', 'description', 'status', 'priority', 'due_date']
    
    def get_queryset(self):
        return Task.objects.filter(
            created_by=self.request.user,
            is_deleted=False
        )
    
    def form_valid(self, form):
        """Handle AJAX form submission"""
        # Track changes
        old_task = Task.objects.get(pk=self.object.pk)
        self.object = form.save()
        
        # Return JSON response with changes
        response_data = {
            'status': 'success',
            'message': 'Task updated successfully',
            'task': {
                'id': self.object.id,
                'title': self.object.title,
                'status': self.object.status,
                'priority': self.object.priority,
                'updated_at': self.object.updated_at.isoformat(),
            }
        }
        
        # Add change tracking
        changes = []
        if old_task.status != self.object.status:
            changes.append(f"Status changed from {old_task.status} to {self.object.status}")
        
        if changes:
            response_data['changes'] = changes
        
        return JsonResponse(response_data)
    
    def form_invalid(self, form):
        """Handle AJAX form errors"""
        return JsonResponse({
            'status': 'error',
            'message': 'Please correct the errors below',
            'errors': form.errors
        }, status=400)
```

### 📝 HTML + JavaScript Examples

#### HTML Template with AJAX

```html
<!-- task_list.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Task Manager</title>
    <script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
</head>
<body>
    <div id="app">
        <h1>My Tasks</h1>
        
        <!-- Task List Container -->
        <div id="task-list">
            <!-- Tasks will be loaded here -->
        </div>
        
        <!-- Create Task Form -->
        <form id="task-form">
            <input type="text" id="title" placeholder="Task title" required>
            <textarea id="description" placeholder="Description"></textarea>
            <select id="priority">
                <option value="low">Low</option>
                <option value="medium">Medium</option>
                <option value="high">High</option>
            </select>
            <button type="submit">Add Task</button>
        </form>
    </div>

    <script>
        // API Base URL
        const API_BASE = '/api/tasks/';
        
        // Load tasks
        async function loadTasks() {
            try {
                const response = await axios.get(API_BASE, {
                    headers: {
                        'Accept': 'application/json'
                    }
                });
                
                renderTasks(response.data.tasks);
            } catch (error) {
                console.error('Error loading tasks:', error);
            }
        }
        
        // Render tasks
        function renderTasks(tasks) {
            const taskList = document.getElementById('task-list');
            taskList.innerHTML = '';
            
            tasks.forEach(task => {
                const taskEl = document.createElement('div');
                taskEl.className = 'task-item';
                taskEl.innerHTML = `
                    <h3>${task.title}</h3>
                    <p>${task.description || ''}</p>
                    <span class="status ${task.status}">${task.status}</span>
                    <span class="priority ${task.priority}">${task.priority}</span>
                    <button onclick="deleteTask(${task.id})">Delete</button>
                `;
                taskList.appendChild(taskEl);
            });
        }
        
        // Create task
        document.getElementById('task-form').addEventListener('submit', async (e) => {
            e.preventDefault();
            
            const formData = {
                title: document.getElementById('title').value,
                description: document.getElementById('description').value,
                priority: document.getElementById('priority').value,
            };
            
            try {
                const response = await axios.post(API_BASE, formData, {
                    headers: {
                        'Content-Type': 'application/json',
                        'X-CSRFToken': getCookie('csrftoken')
                    }
                });
                
                if (response.data.status === 'success') {
                    // Clear form
                    document.getElementById('task-form').reset();
                    // Reload tasks
                    loadTasks();
                    // Show success message
                    showMessage(response.data.message);
                }
            } catch (error) {
                console.error('Error creating task:', error);
                if (error.response) {
                    showMessage(error.response.data.message || 'Error creating task');
                }
            }
        });
        
        // Delete task
        async function deleteTask(taskId) {
            if (!confirm('Are you sure you want to delete this task?')) {
                return;
            }
            
            try {
                await axios.delete(`${API_BASE}${taskId}/`, {
                    headers: {
                        'X-CSRFToken': getCookie('csrftoken')
                    }
                });
                
                loadTasks();
                showMessage('Task deleted successfully');
            } catch (error) {
                console.error('Error deleting task:', error);
                showMessage('Error deleting task');
            }
        }
        
        // Helper functions
        function showMessage(message) {
            const messageEl = document.createElement('div');
            messageEl.className = 'message';
            messageEl.textContent = message;
            document.body.appendChild(messageEl);
            
            setTimeout(() => {
                messageEl.remove();
            }, 3000);
        }
        
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
        
        // Load tasks on page load
        loadTasks();
    </script>
</body>
</html>
```

### 🔄 URL Configuration

```python
# urls.py
from django.urls import path
from .views import (
    TaskListAPIView, TaskDetailAPIView,
    TaskCreateAPIView, TaskUpdateAPIView
)

urlpatterns = [
    # API endpoints
    path('api/tasks/', TaskListAPIView.as_view(), name='task-list-api'),
    path('api/tasks/<int:pk>/', TaskDetailAPIView.as_view(), name='task-detail-api'),
    path('api/tasks/create/', TaskCreateAPIView.as_view(), name='task-create-api'),
    path('api/tasks/<int:pk>/update/', TaskUpdateAPIView.as_view(), name='task-update-api'),
    
    # HTML page (serves the frontend)
    path('tasks/', TemplateView.as_view(template_name='task_list.html'), name='task-page'),
]
```

### 🎯 Benefits of HTML + AJAX Approach

#### ✅ Advantages
- **Better separation**: Frontend handles UI, backend handles data
- **Single Page Applications**: No page reloads needed
- **Mobile friendly**: Same API works for mobile apps
- **Progressive enhancement**: Works without JavaScript too
- **Better performance**: Only transfer data, not HTML

#### 🔧 Implementation Tips
- Use `Accept: application/json` header to detect AJAX requests
- Return consistent JSON format with status, message, and data
- Handle CSRF tokens for POST requests
- Provide fallback HTML rendering for non-JavaScript users
- Use HTTP status codes appropriately (200, 400, 404, 500)

### 📚 Complete API Example

```python
# views.py - Complete API implementation
from django.http import JsonResponse
from django.views.generic import View, ListView, DetailView, CreateView, UpdateView, DeleteView
from django.contrib.auth.mixins import LoginRequiredMixin
from .models import Task

class TaskAPIBase(LoginRequiredMixin, View):
    """Base class for all API endpoints"""
    
    def dispatch(self, request, *args, **kwargs):
        # Check if this is an API request
        if request.headers.get('Accept') == 'application/json':
            return super().dispatch(request, *args, **kwargs)
        
        # Return HTML page for non-API requests
        return JsonResponse({
            'status': 'error',
            'message': 'API endpoint. Use Accept: application/json header'
        }, status=400)

class TaskListAPI(TaskAPIBase, ListView):
    """GET /api/tasks/ - List all tasks"""
    model = Task
    
    def get_queryset(self):
        return Task.objects.filter(
            created_by=self.request.user,
            is_deleted=False
        ).values('id', 'title', 'description', 'status', 'priority', 'created_at')
    
    def get(self, request, *args, **kwargs):
        tasks = list(self.get_queryset())
        
        # Format dates
        for task in tasks:
            task['created_at'] = task['created_at'].isoformat()
        
        return JsonResponse({
            'status': 'success',
            'data': tasks,
            'count': len(tasks)
        })

class TaskDetailAPI(TaskAPIBase, DetailView):
    """GET /api/tasks/<id>/ - Get single task"""
    model = Task
    
    def get_queryset(self):
        return Task.objects.filter(
            created_by=self.request.user,
            is_deleted=False
        )
    
    def get(self, request, *args, **kwargs):
        task = self.get_object()
        
        return JsonResponse({
            'status': 'success',
            'data': {
                'id': task.id,
                'title': task.title,
                'description': task.description,
                'status': task.status,
                'priority': task.priority,
                'created_at': task.created_at.isoformat(),
                'due_date': task.due_date.isoformat() if task.due_date else None,
                'is_completed': task.is_completed,
            }
        })

class TaskCreateAPI(TaskAPIBase, CreateView):
    """POST /api/tasks/ - Create new task"""
    model = Task
    fields = ['title', 'description', 'priority', 'due_date']
    
    def form_valid(self, form):
        self.object = form.save(commit=False)
        self.object.created_by = self.request.user
        self.object.save()
        
        return JsonResponse({
            'status': 'success',
            'message': 'Task created successfully',
            'data': {
                'id': self.object.id,
                'title': self.object.title,
                'status': self.object.status,
                'created_at': self.object.created_at.isoformat(),
            }
        })
    
    def form_invalid(self, form):
        return JsonResponse({
            'status': 'error',
            'message': 'Validation failed',
            'errors': form.errors
        }, status=400)

class TaskUpdateAPI(TaskAPIBase, UpdateView):
    """PUT/PATCH /api/tasks/<id>/ - Update task"""
    model = Task
    fields = ['title', 'description', 'status', 'priority', 'due_date']
    
    def get_queryset(self):
        return Task.objects.filter(
            created_by=self.request.user,
            is_deleted=False
        )
    
    def form_valid(self, form):
        self.object = form.save()
        
        return JsonResponse({
            'status': 'success',
            'message': 'Task updated successfully',
            'data': {
                'id': self.object.id,
                'title': self.object.title,
                'status': self.object.status,
                'updated_at': self.object.updated_at.isoformat(),
            }
        })
    
    def form_invalid(self, form):
        return JsonResponse({
            'status': 'error',
            'message': 'Validation failed',
            'errors': form.errors
        }, status=400)

class TaskDeleteAPI(TaskAPIBase, DeleteView):
    """DELETE /api/tasks/<id>/ - Delete task"""
    model = Task
    success_url = '/api/tasks/'
    
    def get_queryset(self):
        return Task.objects.filter(
            created_by=self.request.user,
            is_deleted=False
        )
    
    def delete(self, request, *args, **kwargs):
        self.object = self.get_object()
        task_title = self.object.title
        
        # Soft delete
        self.object.is_deleted = True
        self.object.deleted_at = timezone.now()
        self.object.save()
        
        return JsonResponse({
            'status': 'success',
            'message': f'Task "{task_title}" deleted successfully'
        })
```

This approach gives you the best of both worlds - clean CBV code on the backend and modern, responsive frontend with AJAX!

---

## Common Mistakes to Avoid

### Mistake 1: Not Specifying model or queryset
**Error**: `ImproperlyConfigured: ... must define 'queryset' or 'model'`
**Solution**: Always specify either model or queryset
```python
# Good
class TaskListView(ListView):
    model = Task

# Bad
class TaskListView(ListView):
    pass  # Missing model!
```

### Mistake 2: Forgetting to Set success_url
**Error**: `ImproperlyConfigured: No URL to redirect to`
**Solution**: Set success_url or implement get_success_url()
```python
# Good
class TaskCreateView(CreateView):
    model = Task
    fields = ['title']
    success_url = reverse_lazy('task_list')

# Bad
class TaskCreateView(CreateView):
    model = Task
    fields = ['title']  # Missing success_url!
```

### Mistake 3: Using reverse() Instead of reverse_lazy()
**Error**: `AppRegistryNotReady` or circular import
**Solution**: Use reverse_lazy() in class attributes
```python
# Good
success_url = reverse_lazy('task_list')

# Bad
success_url = reverse('task_list')  # May cause errors!
```

### Mistake 4: Not Calling super() in Overridden Methods
**Error**: Missing functionality from parent class
**Solution**: Always call super() when overriding methods
```python
# Good
def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)
    context['extra'] = 'data'
    return context

# Bad
def get_context_data(self, **kwargs):
    return {'extra': 'data'}  # Lost all parent context!
```

### Mistake 5: Modifying queryset Directly Instead of Using get_queryset()
**Error**: Changes affect all instances, not just current request
**Solution**: Override get_queryset() for dynamic filtering
```python
# Good
def get_queryset(self):
    return Task.objects.filter(created_by=self.request.user)

# Bad
queryset = Task.objects.all()  # Static, can't access request!
```

---

**You've mastered Django Generic Class-Based Views! 🎉**
