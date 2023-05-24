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
- Add `CSRF_TRUSTED_ORIGINS=['https://*.yourdomain.com']` to settings.py
  
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

```
from rest_framework import generics, permissions
from .serializers import TodoSerializer
from todo.models import Todo

class TodoListCreate(generics.ListCreateAPIView):
  ...
  serializer_class = TodoSerializer
  permission_classes = [permissions.IsAuthenticated]
  ...
```

## Other CRUD Operations
- Go to `/api/urls.py` and add the path below
```
  urlpatterns = [
    path('todos/', views.TodoListCreate.as_view()),
    path('todos/<int:pk>', views.TodoRetrieveUpdateDestroy.as_view()),
  ]
```
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

```
  urlpatterns = [
    path('todos/', views.TodoListCreate.as_view()),
    path('todos/<int:pk>', views.TodoRetrieveUpdateDestroy.as_view()),
    path('todos/<int:pk>/complete', views.TodoToggleComplete.as_view())
  ]
```

- Go to `/api/views.py` and add the code below...

```
...
from .serializers import TodoSerializer, TodoToggleCompleteSerializer
...
class TodoToggleComplete(generics.UpdateAPIView):
  serializer_class = TodoToggleCompleteSerializer
  permission_classes = [permissions.IsAuthenticated]
  
  def get_queryset(self):
    user = self.request.user
    return Todo.objects.filter(user=user)
    
  def perform_update(self, serializer):
    serializer.instance.completed=not(serializer.instance.completed)
    serializer.save()
```

- Go to `/api/serializers.py` and add the code below...

```
class TodoToggleCompleteSerializer(serializers.ModelSerializer):
  class Meta:
    model = Todo
    fields = ['id']
    read_only_fields = ['title', 'memo', 'created', 'completed']
```

## Authentication - Sign Up
- Go to `/backend/settings.py` and add the authtoken app to the `INSTALLED_APPS`

```
INSTALLED_APPS = [
  ...
  'todo',
  'rest_framework',
  'api',
  'rest_framework.authtoken',
]
```

- `python3 manage.py migrate`
- Add the code below in `/backend/settings.py`

```
...
REST_FRAMEWORK = {
  'DEFAULT_AUTHENTICATION_CLASSES':[
    'rest_framework.authentication.TokenAuthentication',
  ]
}
```

- Go to `/api/urls.py` and add the signup endpoint

```
urlpatterns = [
  ...
  path('signup/', views.signup),
]
```

- Go to `/api/views.py` and modify as below

```
...
from todo.models import Todo
from django.db import IntegrityError
from django.contrib.auth.models import User
from rest_framework.parsers import JSONParser
from rest_framework.authtoken.models import Token
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
...
class TodoToggleComplete(generics.UpdateAPIView):
...
@csrf_exempt
def signup(request):
  if request.method == 'POST':
    try:
      data  = JSONParser().parse(request)
      user = User.objects.create_user(
        username=data['username'],
        password=data['password']
      )
      user.save()
      
      token = Token.objects.create(user=user)
      return JsonResponse({'token':str(token)}, status=201)
    except IntegrityError:
      return JsonResponse(
        {'error':'Username taken. Please choose another.'},
        status=400
      )
```

## Authentication - Log In Tokens
- Go to `/api/urls.py` and add the path

```
urlpatterns = [
  ...
  path('login/', views.login),
]
```

- Go to `/api/views.py` to add the login view...

```
...
from django.views.decorators.csrf import csrf_exempt
from django.contrib.auth import authenticate

...

@csrf_exempt
def login(request):
  if request.method=='POST':
    data = JSONParser().parse(request)
    user = authenticate(
      request,
      username=data['username'],
      password=data['password']
    )
    if user is None:
      return JsonResponse(
        {'error':'Unable to login. Check username and password'},
        status=400
      )
    else # Return user token
      try:
        token = Token.objects.get(user=user)
      except: # If token not in db, create a new one
        token = Token.objects.create(user=user)
      
      return JsonResponse({'token':str(token)}, status=201)
    
```
