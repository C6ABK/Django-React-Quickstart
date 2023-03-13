# Django React Quickstart
## Installing Django
- `pip3 install django`
- `mkdir todoapp`
- cd to that folder.
- `python3 -m django startproject backend . `
  - The dot at the end will create the django project in that directory.

## Notes for codeanywhere IDE
- Go into settings.py and allow all hosts
- `python manage.py runserver 0.0.0.0:8000`
- Add `CSRF_TRUSTED_ORIGINS=['https://*yourdomain.com']` to settings.py
  
## Creating Backend Server
- `python3 manage.py startapp todo`
- Add to `INSTALLED_APPS` in `settings.py`

```
INSTALLED_APPS = [
    ...\
    'django.contrib.staticfiles',
    'todo',
  ]
```

### Open /todo/models.py and modify as below
```
from django.db import models
from django.contrib.auth.models import User

class Todo(models.Model):
  title = models.CharField(max_length=100)
  memo = models.TextField(blank=True)
  
  # set to current time
  created = models.DateTimeField(auto_now_add=True)
  completed = models.BooleanField(default=False)
  
  # user who posted this
  user = models.ForeignKey(User, on_delete=models.CASCADE)
  
  def __str__(self):
    return self.title
```

- `python3 manage.py makemigrations`
- `python3 manage.py migrate`
- `python3 manage.py createsuperuser`
- Run the server and test the django admin login

### Add the Todo model to admin
- Go to `.../todo/admin.py`

```
from django.contrib import admin
from .models import Todo

admin.site.register(Todo)
```
- Test the admin login again and the Todo functions

## Serialization
- `pip3 install djangorestframework`
- Add `rest_framework` to your `INSTALLED_APPS` in `../backend/settings.py`

```
INSTALLED_APPS = [
  ...
  'todo',
  'rest_framework',
]
```

### Adding the API app
- `python manage.py startapp api`
- Add to `INSTALLED_APPS`

```
INSTALLED_APPS = [
  ...
  'todo',
  'rest_framework',
  'api',
]
```

- In the `api` folder, create `serializers.py` and fill as below...

```
from rest_framework import serializers
from todo.models import Todo

class TodoSerializer(serializers.ModelSerializer):
  created = serializers.ReadOnlyField()
  completed = serializers.ReadOnlyField()
  
  class Meta:
    model = Todo
    fields = ['id', 'title', 'memo', 'created', 'completed']
```

- Under `class Meta` we specify our database model `Todo` and the fields we want to expose. Fields not specified here will not be exposed in the API.

## URLS and Class-based views
- Add the code below to `/backend/urls.py`

```
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
  path('admin/', admin.site.urls),
  path('api/', include('api.urls')),
]
```

- in the `api` folder, create `urls.py` and add the code below...

```
from django.urls import path
from .import views

urlpatterns = [
  path('todos/', views.TodoList.as_view()),
]
```

- Replace the code in `/api/views.py` with the following...

```
from rest_framework import generics
from .serializers import TodoSerializer
from todo.models import Todo

class TodoList(generics.ListAPIView):
  serializer_class = TodoSerializer
  
  def get_queryset(self):
    user = self.request.user
    return Todo.objects.filter(user=user).order_by('-created')
```

## Creating Todos via the ListCreateAPIView
The `ListAPIView` gives us a read-only endpoint of todo model instances. We can use `ListCreateAPIView` to allow a read-write endpoint.

- Go to `/api/views.py` and modify as below...

```
class TodoListCreate(generics.ListCreateAPIView):
...
```

- Go to `/api/urls.py` and modify as below...

```
...
urlpatterns = [
   path('todos/', views.TodoListCreate.as_view()),
]
```

- We haven't specified a user to create a todo yet so attempting to add a todo at this stage would cause an integrity error.

- Go to `/api/views.py` and add the code below...

```
...
def perform_create(self, serializer):
  serializer.save(user=self.request.user)
```

## Permissions
- Go to `/api/views.py` and modify as below

<pre>
from rest_framework import generics, <b>permissions</b>
from .serializers import TodoSerializer
from todo.models import Todo

class TodoListCreate(generics.ListCreateAPIView):
  ...
  serializer_class = TodoSerializer
  <b>permission_classes = [permissions.IsAuthenticated]</b>
  ...
</pre>

## Other CRUD Operations
- Go to `/api/urls.py` and add the path below
<pre>
  urlpatterns = [
    path('todos/', views.TodoListCreate.as_view()),
    <b>path('todos/<int:pk>', views.TodoRetrieveUpdateDestroy.as_view()),</b>
  ]
</pre>
- Go to `/api/views.py` and add the TodoRetrieveUpdateDestroy generic view...
```
...
class TodoListCreate(generics.ListCreateAPIView):
...
class TodoRetrieveUpdateDestroy(generics.RetrieveUpdateDestroyAPIView):
  serializer_class = TodoSerializer
  permission_classes = [permissions.IsAuthenticated]
  
  def get_queryset(self):
    user = self.request.user
    # User can only update, delete own posts
    return Todo.objects.filter(user=user)
```
## Completing a Todo
- Go to `/api/urls.py` and add the path below...
<pre>
  urlpatterns = [
    path('todos/', views.TodoListCreate.as_view()),
    path('todos/<int:pk>', views.TodoRetrieveUpdateDestroy.as_view()),
    <b>path('todos/<int:pk>/complete', views.TodoToggleComplete.as_view())</b>
  ]
</pre>
