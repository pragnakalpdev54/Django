# Part 2: Function-Based Views - Models & Database

## Understanding Django Models

Models are Python classes that define the structure of your database tables. Django's ORM (Object-Relational Mapping) automatically converts these Python classes into database tables.

### Why Use Django ORM?

- **Database Agnostic**: Write Python, not SQL
- **Type Safety**: Python type checking
- **Relationships**: Easy foreign keys and many-to-many
- **Migrations**: Automatic schema versioning
- **Query API**: Powerful and intuitive querying

---

## Creating the Task Model

For our Task Management System, we need a Task model with the following features:
- Title and description
- Priority levels (Low, Medium, High)
- Status tracking (Pending, In Progress, Completed)
- User ownership
- Timestamps
- Due dates

### Step 1: Define the Model

Open `tasks/models.py` and add:

```python
from django.db import models
from django.contrib.auth.models import User
from django.utils import timezone

class Task(models.Model):
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
    
    title = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    priority = models.CharField(
        max_length=10, 
        choices=PRIORITY_CHOICES, 
        default='medium'
    )
    status = models.CharField(
        max_length=20, 
        choices=STATUS_CHOICES, 
        default='pending'
    )
    created_by = models.ForeignKey(
        User, 
        on_delete=models.CASCADE, 
        related_name='tasks'
    )
    created_at = models.DateTimeField(default=timezone.now)
    updated_at = models.DateTimeField(auto_now=True)
    due_date = models.DateField(null=True, blank=True)
    is_completed = models.BooleanField(default=False)
    
    # Soft delete fields
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        ordering = ['-created_at']
        verbose_name = 'Task'
        verbose_name_plural = 'Tasks'
    
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

### Understanding Model Fields

**CharField**
```python
title = models.CharField(max_length=200)
```
- For short text (up to max_length characters)
- Required by default
- Use for: names, titles, short descriptions

**TextField**
```python
description = models.TextField(blank=True)
```
- For long text (unlimited length)
- `blank=True`: Field can be empty in forms
- Use for: descriptions, comments, content

**ForeignKey**
```python
created_by = models.ForeignKey(User, on_delete=models.CASCADE, related_name='tasks')
```
- Creates many-to-one relationship
- `on_delete=models.CASCADE`: Delete tasks when user is deleted
- `related_name='tasks'`: Access user's tasks via `user.tasks.all()`

**DateTimeField**
```python
created_at = models.DateTimeField(default=timezone.now)
updated_at = models.DateTimeField(auto_now=True)
```
- `default=timezone.now`: Set on creation
- `auto_now=True`: Update on every save
- Always use `timezone.now` (not `timezone.now()`)

**DateField**
```python
due_date = models.DateField(null=True, blank=True)
```
- Stores date only (no time)
- `null=True`: Can be NULL in database
- `blank=True`: Can be empty in forms

**BooleanField**
```python
is_completed = models.BooleanField(default=False)
```
- True/False values
- Always has a default value

**Choices**
```python
PRIORITY_CHOICES = [
    ('low', 'Low'),      # (stored value, display value)
    ('medium', 'Medium'),
    ('high', 'High'),
]
priority = models.CharField(max_length=10, choices=PRIORITY_CHOICES)
```
- Restricts field to predefined values
- First value stored in DB, second shown to users
- Access display value: `task.get_priority_display()`

### Meta Class Options

```python
class Meta:
    ordering = ['-created_at']  # Default ordering (newest first)
    verbose_name = 'Task'       # Singular name in admin
    verbose_name_plural = 'Tasks'  # Plural name in admin
```

Other useful Meta options:
```python
class Meta:
    db_table = 'custom_table_name'  # Custom table name
    unique_together = ['field1', 'field2']  # Unique combination
    indexes = [models.Index(fields=['title'])]  # Database indexes
```

### Model Methods

```python
def __str__(self):
    return self.title
```
- Returns string representation
- Used in admin interface and shell
- Should return something meaningful

```python
def mark_complete(self):
    self.is_completed = True
    self.status = 'completed'
    self.save()
```
- Custom business logic
- Encapsulates related operations
- Makes code more maintainable

---

## Database Migrations

Migrations are Django's way of propagating model changes to the database schema.

### Step 1: Create Migrations

```bash
python manage.py makemigrations
```

**Output:**
```
Migrations for 'tasks':
  tasks/migrations/0001_initial.py
    - Create model Task
```

This creates a migration file in `tasks/migrations/0001_initial.py`

### Step 2: View Migration SQL

```bash
python manage.py sqlmigrate tasks 0001
```

This shows the SQL that will be executed:

```sql
BEGIN;
CREATE TABLE "tasks_task" (
    "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
    "title" varchar(200) NOT NULL,
    "description" text NOT NULL,
    "priority" varchar(10) NOT NULL,
    "status" varchar(20) NOT NULL,
    "created_at" datetime NOT NULL,
    "updated_at" datetime NOT NULL,
    "due_date" date NULL,
    "is_completed" bool NOT NULL,
    "created_by_id" integer NOT NULL REFERENCES "auth_user" ("id")
);
CREATE INDEX "tasks_task_created_by_id" ON "tasks_task" ("created_by_id");
COMMIT;
```

### Step 3: Apply Migrations

```bash
python manage.py migrate
```

**Output:**
```
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions, tasks
Running migrations:
  Applying tasks.0001_initial... OK
```

### Migration Commands Summary

```bash
# Create migration files
python manage.py makemigrations

# Create migration for specific app
python manage.py makemigrations tasks

# Name your migration
python manage.py makemigrations --name add_task_category

# Show all migrations
python manage.py showmigrations

# Show SQL for migration
python manage.py sqlmigrate tasks 0001

# Apply migrations
python manage.py migrate

# Migrate to specific migration
python manage.py migrate tasks 0001

# Reverse migrations
python manage.py migrate tasks zero
```

---

## Django ORM: Querying the Database

The Django ORM provides a powerful API for database operations.

### Using Django Shell

```bash
python manage.py shell
```

### Creating Objects

**Method 1: Create and Save**
```python
from tasks.models import Task
from django.contrib.auth.models import User

# Get a user
user = User.objects.first()

# Create task
task = Task(
    title="Complete Django Tutorial",
    description="Learn Django from scratch",
    priority="high",
    created_by=user
)
task.save()
```

**Method 2: Create in One Step**
```python
task = Task.objects.create(
    title="Build Portfolio",
    description="Create personal portfolio website",
    priority="medium",
    created_by=user
)
```

**Method 3: Get or Create**
```python
task, created = Task.objects.get_or_create(
    title="Daily Standup",
    created_by=user,
    defaults={'priority': 'low'}
)
# created is True if new, False if existed
```

### Reading Objects

**Get All Objects**
```python
# Get all tasks
all_tasks = Task.objects.all()

# Iterate over tasks
for task in all_tasks:
    print(task.title)
```

**Get Single Object**
```python
# Get by ID
task = Task.objects.get(id=1)

# Get by field
task = Task.objects.get(title="Complete Django Tutorial")

# Raises DoesNotExist if not found
# Raises MultipleObjectsReturned if multiple found
```

**Filter Objects**
```python
# Filter by field
high_priority = Task.objects.filter(priority='high')

# Multiple conditions (AND)
pending_high = Task.objects.filter(
    priority='high',
    status='pending'
)

# Exclude
not_completed = Task.objects.exclude(is_completed=True)

# Chain filters
tasks = Task.objects.filter(priority='high').exclude(status='completed')
```

**Field Lookups**
```python
# Exact match (default)
Task.objects.filter(title__exact="My Task")

# Case-insensitive exact
Task.objects.filter(title__iexact="my task")

# Contains
Task.objects.filter(title__contains="Django")

# Case-insensitive contains
Task.objects.filter(title__icontains="django")

# Starts with
Task.objects.filter(title__startswith="Complete")

# Greater than
Task.objects.filter(id__gt=5)

# Less than or equal
Task.objects.filter(id__lte=10)

# In list
Task.objects.filter(priority__in=['high', 'medium'])

# Date comparisons
from datetime import date
Task.objects.filter(due_date__gte=date.today())
```

**Ordering**
```python
# Order by field (ascending)
Task.objects.order_by('created_at')

# Descending
Task.objects.order_by('-created_at')

# Multiple fields
Task.objects.order_by('priority', '-created_at')
```

**Limiting Results**
```python
# First 5 tasks
Task.objects.all()[:5]

# Tasks 5-10
Task.objects.all()[5:10]

# First task
Task.objects.first()

# Last task
Task.objects.last()
```

### Updating Objects

**Update Single Object**
```python
task = Task.objects.get(id=1)
task.title = "Updated Title"
task.priority = "high"
task.save()
```

**Update Multiple Objects**
```python
# Update all pending tasks
Task.objects.filter(status='pending').update(priority='medium')

# Returns number of rows updated
count = Task.objects.filter(is_completed=False).update(status='in_progress')
```

### Deleting Objects

**Delete Single Object**
```python
task = Task.objects.get(id=1)
task.delete()
```

**Delete Multiple Objects**
```python
# Delete all completed tasks
Task.objects.filter(is_completed=True).delete()

# Delete all tasks
Task.objects.all().delete()
```

### Complex Queries with Q Objects

```python
from django.db.models import Q

# OR condition
Task.objects.filter(Q(priority='high') | Q(status='in_progress'))

# AND with OR
Task.objects.filter(
    Q(priority='high') | Q(priority='medium'),
    status='pending'
)

# NOT
Task.objects.filter(~Q(status='completed'))

# Complex combinations
Task.objects.filter(
    (Q(priority='high') & Q(status='pending')) |
    (Q(priority='medium') & Q(is_completed=False))
)
```

### Aggregation and Annotation

```python
from django.db.models import Count, Avg, Max, Min, Sum

# Count tasks
task_count = Task.objects.count()

# Count by filter
pending_count = Task.objects.filter(status='pending').count()

# Aggregate
stats = Task.objects.aggregate(
    total=Count('id'),
    completed=Count('id', filter=Q(is_completed=True))
)
# Returns: {'total': 10, 'completed': 5}

# Annotate (add calculated field to each object)
from django.contrib.auth.models import User
users_with_task_count = User.objects.annotate(
    task_count=Count('tasks')
)
for user in users_with_task_count:
    print(f"{user.username}: {user.task_count} tasks")
```

### Related Object Queries

```python
# Get user's tasks (forward relation)
user = User.objects.get(username='john')
user_tasks = user.tasks.all()  # uses related_name='tasks'

# Filter related objects
high_priority_tasks = user.tasks.filter(priority='high')

# Reverse lookup (without related_name)
# user.task_set.all()

# Select related (optimize queries)
tasks = Task.objects.select_related('created_by').all()
# Fetches user data in same query (reduces DB hits)

# Prefetch related (for many-to-many or reverse FK)
users = User.objects.prefetch_related('tasks').all()
```

---

## Register Model in Admin

To manage tasks through Django admin, register the model.

Open `tasks/admin.py`:

```python
from django.contrib import admin
from .models import Task

@admin.register(Task)
class TaskAdmin(admin.ModelAdmin):
    list_display = [
        'title', 
        'priority', 
        'status', 
        'created_by', 
        'due_date', 
        'is_completed',
        'created_at'
    ]
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
    
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        return qs.select_related('created_by')
```

### Admin Customization Options

**list_display**: Columns in list view
**list_filter**: Sidebar filters
**search_fields**: Fields to search
**list_editable**: Edit directly in list view
**date_hierarchy**: Date-based navigation
**readonly_fields**: Cannot be edited
**fieldsets**: Group fields in edit form
**ordering**: Default ordering
**list_per_page**: Pagination

### Access Admin Interface

1. Start server: `python manage.py runserver`
2. Visit: `http://127.0.0.1:8000/admin/`
3. Login with superuser credentials
4. You'll see Tasks in the admin!

---

## Model Relationships

Django supports three types of relationships:

### One-to-Many (ForeignKey)

```python
class Task(models.Model):
    created_by = models.ForeignKey(User, on_delete=models.CASCADE)
```

**on_delete options:**
- `CASCADE`: Delete tasks when user deleted
- `PROTECT`: Prevent user deletion if has tasks
- `SET_NULL`: Set to NULL (requires null=True)
- `SET_DEFAULT`: Set to default value
- `DO_NOTHING`: Do nothing (can cause DB errors)

### Many-to-Many

```python
class Task(models.Model):
    tags = models.ManyToManyField('Tag', blank=True)

class Tag(models.Model):
    name = models.CharField(max_length=50)
```

**Usage:**
```python
task = Task.objects.get(id=1)
tag = Tag.objects.get(name='urgent')

# Add tag
task.tags.add(tag)

# Remove tag
task.tags.remove(tag)

# Get all tags
task.tags.all()

# Clear all tags
task.tags.clear()
```

### One-to-One

```python
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    bio = models.TextField()
    avatar = models.ImageField(upload_to='avatars/')
```

---

## Database Setup

Django supports multiple databases:
- SQLite (default, good for development)
- **PostgreSQL (recommended for production and this tutorial)**
- MySQL
- Oracle

### Using PostgreSQL

PostgreSQL is a powerful, open-source relational database. We'll use it for our project.

**Step 1: Install PostgreSQL**

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install postgresql postgresql-contrib

# macOS (using Homebrew)
brew install postgresql

# Windows
# Download installer from https://www.postgresql.org/download/windows/
```

**Step 2: Install Python PostgreSQL Adapter**

```bash
pip install psycopg2-binary
```

**Step 3: Create Database and User**

```bash
# Access PostgreSQL shell
sudo -u postgres psql

# In PostgreSQL shell, run:
CREATE DATABASE taskmanager_db;
CREATE USER taskmanager_user WITH PASSWORD 'your_secure_password';
ALTER ROLE taskmanager_user SET client_encoding TO 'utf8';
ALTER ROLE taskmanager_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE taskmanager_user SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE taskmanager_db TO taskmanager_user;

# Exit PostgreSQL shell
\q
```

**Step 4: Configure Django Settings**

Update `taskmanager/settings.py`:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'taskmanager_db',
        'USER': 'taskmanager_user',
        'PASSWORD': 'your_secure_password',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

**Best Practice: Use Environment Variables**

Install python-decouple:
```bash
pip install python-decouple
```

Create `.env` file in project root:
```env
DB_NAME=taskmanager_db
DB_USER=taskmanager_user
DB_PASSWORD=your_secure_password
DB_HOST=localhost
DB_PORT=5432
```

Update `settings.py`:
```python
from decouple import config

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
```

**Step 5: Test Connection**

```bash
python manage.py check
```

If successful, you'll see: "System check identified no issues."

---

## Best Practices

### 1. Always Use Migrations
```bash
# After any model change
python manage.py makemigrations
python manage.py migrate
```

### 2. Use __str__ Method
```python
def __str__(self):
    return self.title
```

### 3. Add Indexes for Frequently Queried Fields
```python
class Meta:
    indexes = [
        models.Index(fields=['status', 'priority']),
    ]
```

### 4. Use select_related and prefetch_related
```python
# Optimize queries
tasks = Task.objects.select_related('created_by').all()
```

### 5. Validate Data in Models
```python
from django.core.exceptions import ValidationError

def clean(self):
    if self.due_date and self.due_date < timezone.now().date():
        raise ValidationError('Due date cannot be in the past')
```

### 6. Use Choices for Fixed Options
```python
PRIORITY_CHOICES = [('low', 'Low'), ('medium', 'Medium'), ('high', 'High')]
```

---

## Summary

You've learned:

✅ How to create Django models
✅ Understanding model fields and options
✅ Creating and applying migrations
✅ Django ORM query API
✅ CRUD operations
✅ Complex queries with Q objects
✅ Model relationships
✅ Admin interface customization

### Next Steps

👉 **Continue to:** [`03_FBV_Views_and_URLs.md`](03_FBV_Views_and_URLs.md)

In the next section, we'll:
- Create function-based views
- Handle HTTP requests
- Configure URL routing
- Pass data to templates
- Implement CRUD operations

---

**Your database is ready! Let's build the views! 🎯**
