# Exam
 

To carry out the project we will need to install Redis:
Redis (5.0.6)
```
$ wget http://download.redis.io/releases/redis-5.0.7.tar.gz
$ tar xzf redis-5.0.6.tar.gz
$ cd redis-5.0.6
$ make
```


In the same way we install Python Virtual Env and Dependencies
We create a directory called image_parroter

```
$ mkdir image_parroter
$ cd image_parroter
$ python3 -m venv venv 
$ source venv/bin/activate
```

We install Django, Celery, Pillow, django-widget-tweaks

```
(venv) $ pip3 install Django Celery redis Pillow django-widget-tweaks
(venv) $ pip3 freeze > requirements.txt
```

Django project configuration
We create a project called image_parroter and once inside the project we will create a Django app called thumbnailer

```
(venv) $ django-admin startproject image_parroter
(venv) $ cd image_parroter
(venv) $ python manage.py startapp thumbnailer
```

We see the structure of our project
```
$ tree -I venv
```

Inside image_parroter / image_parroter / image_parroter we will create a celery.py documents
to integrate Celery into the project

### image_parroter/image_parroter/image_parroter/celery.py

```
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'image_parroter.settings')

celery_app = Celery('image_parroter')
celery_app.config_from_object('django.conf:settings', namespace='CELERY')
celery_app.autodiscover_tasks()
```
 
Within our settings.py file we will add the following code at the bottom.

### image_parroter/image_parroter/image_parroter/settings.py
``` 
... skipping to the bottom
 
# celery
CELERY_BROKER_URL = 'redis://localhost:6379'
CELERY_RESULT_BACKEND = 'redis://localhost:6379'
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TASK_SERIALIZER = 'json'
 ```
 
 
 
We will import celery from our main script __init__.py

### image_parroter/image_parroter/image_parroter/__init__.py
``` 
from .celery import celery_app
__all__ = ('celery_app',)
 ```
Inside the thumbnailer application we will create a module called task.py where we will import shared_tasks and define the function adding_task
 
### image_parroter/image_parroter/thumbnailer/tasks.py
 
 ```
from celery import shared_task
 
@shared_task
def adding_task(x, y):
    return x + y
```
We modify our settings.py module again in the INSTALLED_APPS section
 
### image_parroter/image_parroter/image_parroter/settings.py
 
```
... skipping to the INSTALLED_APPS
 
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'thumbnailer.apps.ThumbnailerConfig',
    'widget_tweaks',
]
```

We run our redis server inside the first image_parroter folder

```
$ redis-server
```
```
48621:C 21 May 21:55:23.706 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
48621:C 21 May 21:55:23.707 # Redis version=4.0.8, bits=64, commit=00000000, modified=0, pid=48621, just started
48621:C 21 May 21:55:23.707 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
48621:M 21 May 21:55:23.708 * Increased maximum number of open files to 10032 (it was originally set to 2560).
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 4.0.8 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 48621
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               
 
48621:M 21 May 21:55:23.712 # Server initialized
48621:M 21 May 21:55:23.712 * Ready to accept connections
```
 
In case of showing a port error it is already in use, we will stop the redis server and execute it again

```
$ sudo systemctl stop redis
$ redis-server
```
 
We open a second terminal

```
$ cd image_parroter
$ source venv/bin/activate
(venv) $ celery worker -A image_parroter --loglevel=info
 
 -------------- celery@Adams-MacBook-Pro-191.local v4.3.0 (rhubarb)
---- **** ----- 
--- * ***  * -- Darwin-18.5.0-x86_64-i386-64bit 2019-05-22 03:01:38
-- * - **** --- 
- ** ---------- [config]
- ** ---------- .> app:         image_parroter:0x110b18eb8
- ** ---------- .> transport:   redis://localhost:6379//
- ** ---------- .> results:     redis://localhost:6379/
- *** --- * --- .> concurrency: 8 (prefork)
-- ******* ---- .> task events: OFF (enable -E to monitor tasks in this worker)
--- ***** ----- 
 -------------- [queues]
                .> celery           exchange=celery(direct) key=celery
                
[tasks]
  . thumbnailer.tasks.adding_task
```

We will open a third terminal to initial the Django Python shell and try adding_task

```
$ cd image_parroter
$ source venv/bin/activate
(venv) $ python manage.py shell
Python 3.6.6 |Anaconda, Inc.| (default, Jun 28 2018, 11:07:29) 
>>> from thumbnailer.tasks import adding_task
>>> task = adding_task.delay(2, 5)
>>> print(f"id={task.id}, state={task.state}, status={task.status}") 
id=86167f65-1256-497e-b5d9-0819f24e95bc, state=SUCCESS, status=SUCCESS
>>> task.get()
7
``` 
 
Create thumbnails of images within a celery task
Returning to our task.py module we will import the Image class from the PIL package and add a new task called make_thumbnails that accepts the path of our image and a list of width and height dimensions of 2 tuples to create thumbnails.
 
### image_parroter/image_parroter/thumbnailer/tasks.py
```
import os
from zipfile import ZipFile
 
from celery import shared_task
from PIL import Image
 
from django.conf import settings
 
@shared_task
def make_thumbnails(file_path, thumbnails=[]):
    os.chdir(settings.IMAGES_DIR)
    path, file = os.path.split(file_path)
    file_name, ext = os.path.splitext(file)
 
    zip_file = f"{file_name}.zip"
    results = {'archive_path': f"{settings.MEDIA_URL}images/{zip_file}"}
    try:
        img = Image.open(file_path)
        zipper = ZipFile(zip_file, 'w')
        zipper.write(file)
        os.remove(file_path)
        for w, h in thumbnails:
            img_copy = img.copy()
            img_copy.thumbnail((w, h))
            thumbnail_file = f'{file_name}_{w}x{h}.{ext}'
            img_copy.save(thumbnail_file)
            zipper.write(thumbnail_file)
            os.remove(thumbnail_file)
 
        img.close()
        zipper.close()
    except IOError as e:
        print(e)
 
    return results
 
```
 
We enter the settings.py module to add the MEDIA_ROOT, MEDIA_URL, IMAGES_DIR

### image_parroter/image_parroter/settings.py

```
... skipping down to the static files section
 
# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/2.2/howto/static-files/
 
STATIC_URL = '/static/'
MEDIA_URL = '/media/'
 
MEDIA_ROOT = os.path.abspath(os.path.join(BASE_DIR, 'media'))
IMAGES_DIR = os.path.join(MEDIA_ROOT, 'images')
 
if not os.path.exists(MEDIA_ROOT) or not os.path.exists(IMAGES_DIR):
    os.makedirs(IMAGES_DIR)
```
 
Inside thumbnailer / views.py we will import the django.views.View class and use it to create HomeView that contains the get and post methods
 
### image_parroter/image_parroter/thumbnailer/views.py
```
import os
 
from celery import current_app
 
from django import forms
from django.conf import settings
from django.http import JsonResponse
from django.shortcuts import render
from django.views import View
 
from .tasks import make_thumbnails
 
class FileUploadForm(forms.Form):
    image_file = forms.ImageField(required=True)
 
class HomeView(View):
    def get(self, request):
        form = FileUploadForm()
        return render(request, 'thumbnailer/home.html', { 'form': form })
    
    def post(self, request):
        form = FileUploadForm(request.POST, request.FILES)
        context = {}
 
        if form.is_valid():
            file_path = os.path.join(settings.IMAGES_DIR, request.FILES['image_file'].name)
 
            with open(file_path, 'wb+') as fp:
                for chunk in request.FILES['image_file']:
                    fp.write(chunk)
 
            task = make_thumbnails.delay(file_path, thumbnails=[(128, 128)])
 
            context['task_id'] = task.id
            context['task_status'] = task.status
 
            return render(request, 'thumbnailer/home.html', context)
 
        context['form'] = form
 
        return render(request, 'thumbnailer/home.html', context)
 
 
class TaskView(View):
    def get(self, request, task_id):
        task = current_app.AsyncResult(task_id)
        response_data = {'task_status': task.status, 'task_id': task.id}
 
        if task.status == 'SUCCESS':
            response_data['results'] = task.get()
 
        return JsonResponse(response_data)
```
 
Inside thumbernailer we will create the urls.py module
 
### image_parroter/image_parroter/thumbnailer/urls.py

```
from django.urls import path
 
from . import views
 
urlpatterns = [
  path('', views.HomeView.as_view(), name='home'),
  path('task/<str:task_id>/', views.TaskView.as_view(), name='task'),
]
```
 
In the urls.py module of image_parroter we will add the application level URLs
### image_parroter/image_parroter/urls.py

```
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static
 
urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('thumbnailer.urls')),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
``` 
 
We create a directory to store the unique template within the thumbnail directory
```
(venv) $ mkdir -p thumbnailer/templates/thumbnailer
```
Within this directory we will add home.html in which we will load the widget_tweaks tags and import bulma CSS and a library called Axios.js
 
```
<!-- image_parroter/image_parroter/thumbnailer/templates/thumbnailer/home.html -->
{% load widget_tweaks %}
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Thumbnailer</title>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/bulma/0.7.5/css/bulma.min.css">
  <script src="https://cdn.jsdelivr.net/npm/vue"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/axios/0.18.0/axios.min.js"></script>
  <script defer src="https://use.fontawesome.com/releases/v5.0.7/js/all.js"></script>
</head>
<body>
  <nav class="navbar" role="navigation" aria-label="main navigation">
    <div class="navbar-brand">
      <a class="navbar-item" href="/">
        Thumbnailer
      </a>
    </div>
  </nav>
  <section class="hero is-primary is-fullheight-with-navbar">
    <div class="hero-body">
      <div class="container">
        <h1 class="title is-size-1 has-text-centered">Thumbnail Generator</h1>
        <p class="subtitle has-text-centered" id="progress-title"></p>
        <div class="columns is-centered">
          <div class="column is-8">
            <form action="{% url 'home' %}" method="POST" enctype="multipart/form-data">
              {% csrf_token %}
              <div class="file is-large has-name">
                <label class="file-label">
                  {{ form.image_file|add_class:"file-input" }}
                  <span class="file-cta">
                    <span class="file-icon"><i class="fas fa-upload"></i></span>
                    <span class="file-label">Browse image</span>
                  </span>
                  <span id="file-name" class="file-name" 
                    style="background-color: white; color: black; min-width: 450px;">
                  </span>
                </label>
                <input class="button is-link is-large" type="submit" value="Submit">
              </div>
              
            </form>
          </div>
        </div>
      </div>
    </div>
  </section>
  <script>
  var file = document.getElementById('{{form.image_file.id_for_label}}');
  file.onchange = function() {
    if(file.files.length > 0) {
      document.getElementById('file-name').innerHTML = file.files[0].name;
    }
  };
  </script>
 
  {% if task_id %}
  <script>
  var taskUrl = "{% url 'task' task_id=task_id %}";
  var dots = 1;
  var progressTitle = document.getElementById('progress-title');
  updateProgressTitle();
  var timer = setInterval(function() {
    updateProgressTitle();
    axios.get(taskUrl)
      .then(function(response){
        var taskStatus = response.data.task_status
        if (taskStatus === 'SUCCESS') {
          clearTimer('Check downloads for results');
          var url = window.location.protocol + '//' + window.location.host + response.data.results.archive_path;
          var a = document.createElement("a");
          a.target = '_BLANK';
          document.body.appendChild(a);
          a.style = "display: none";
          a.href = url;
          a.download = 'results.zip';
          a.click();
          document.body.removeChild(a);
        } else if (taskStatus === 'FAILURE') {
          clearTimer('An error occurred');
        }
      })
      .catch(function(err){
        console.log('err', err);
        clearTimer('An error occurred');
      });
  }, 800);
 
  function updateProgressTitle() {
    dots++;
    if (dots > 3) {
      dots = 1;
    }
    progressTitle.innerHTML = 'processing images ';
    for (var i = 0; i < dots; i++) {
      progressTitle.innerHTML += '.';
    }
  }
  function clearTimer(message) {
    clearInterval(timer);
    progressTitle.innerHTML = message;
  }
  </script> 
  {% endif %}
</body>
</html>
 
```

We will open another terminal in the Python virtual environment to join the Django development server
```
(venv) $ python manage.py runserver
```
Once this is done, we will enter the address of our server
http://127.0.0.1:8000/

We look for an image inside our computer

It generates a zip file which contains our original and miniature image
