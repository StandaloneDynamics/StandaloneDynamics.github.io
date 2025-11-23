---
title: "Serving Flutter Web Application With Django"
date: "2025-01-28"
description: "Getting Django to serve flutter applications"
tags: ["Django", "Flutter", "Python", "Dart"]
draft: false
---

# Introduction

I was recently playing with around with [Flutter web](https://flutter.dev/multi-platform/web) and was wondering how I would go about getting it to work with [Django](https://www.djangoproject.com/) and deploying them together.
The application I decided to test this with is a kanban type application, so flutter would be used to build the frontend UI and Django would provide the REST API.


## Development

### Directory Structure
Our project directory structure will looks very similar to the traditional Django project structure.

```
kanban/
  db.sqlite3
  apps/
  kanban/
  manage.py
  flutter/
    build/
	  web/
    pubspec.yaml
    ....
```
So as you can see we have placed the flutter project within the Django project.


### Django
For the REST API, I used [Django Rest Framework](https://www.django-rest-framework.org/), so I had endpoints such as
```
http://localhost:8000/api/v1/projects
http://localhost:8000/api/v1/tasks
```
If you have used django the `urls.py` file would look as follows

```
urlpatterns = [
	path('api/v1/', include(router.urls)),
]
```

where `router.urls` would be the `ViewSets` for the project and task models.

So once that was running this could be served via the usual `./manage.py runserver`

### Flutter

The frontend side is a single page application which closely resembles developing a mobile application in flutter. To run the web application during development
```
flutter run -d web-server --web-port=9090
```
 and any changes made will be hot reloaded automatically.
 
 
## Integrating with Django.

Once our frontend is built, we can build it via

```
flutter build web --release
```
This will build the flutter application with all the required `javascript`,`icons` and any `assert` files and placed in the  `build/web` directory which will look similar to the below

```
flutter/
  ...
  build/
    web/
      assets/
      flutter.js
      flutter_service_worker.js
      index.html
	  ....
```

During the development we were essentially running 2 services.

```
./manage.py runserver
```
and

```
flutter run -d web-server --web-port=9090
```
Now we just want the entire application to be served by Django.


## Index file
We can tell Django about the index file by creating a template view and treating the `index.html` file as a template.
We need to first add the file path of `flutter/build/web` to the Django templates settings
```
FLUTTER_DIR = BASE_DIR / 'flutter/build/web'
TEMPLATES = [
    ...
	DIRS: [FLUTTER_DIR]
	...
]
```
We can create a template view in our apps `views.py` file

```
class IndexView(TemplateView):
    template_name='index.html'
```

And add it to our `urls.py` file
```
path("web/", IndexView.as_view()),
```

If we browse to `http://localhost:8000/web` we'll be presented with a blank page. This due to the fact that all the static assets(css,js) are relative to the index file and don't follow the Django's way of specifying static files.

So now we need a way to serve this files. We could go in a modify the `index.html` file and change the urls to `{% static .... %}` but then we would need to do that each time we rebuild flutter, we are trying to avoid that path.

## Serving Flutter Static Files

### Django Serve

There's a quick way to serve the static files from the flutter application.
We can modify our `index.html` file and change `<base href=/>` to `<base href="/web/>`, what this html tag does is, it sets the base url for all relative paths in the `index.html` file, so `/file.js` would become `/web/file.js`.

You might notice that if we rebuild flutter we'd have to do this again, luckily for us flutter has got a build flag we can use `--base-href`, so now to build our flutter app
```
flutter build web --release --base-href /web/
```
Which will change the `<base href=>` to the appropriate value.

With this change we can serve this file in Django by matching the appropriate url and serving the static files, in `urls.py`
```
path("web/<path:resource>', serve_flutter),
```

The `serve_flutter` function will then serve the file at that url resource.

```
from django.views.static import serve

def serve_flutter(request, resource):
    return serve(request, resource, settings.FLUTTER_DIR)
```

The issue with this is that the `serve` function is only meant to be used for development purposes and not production, we need to find a better way.

### Whitenoise

Whitenoise is a library that allows a python web application to serve its own static files with out the need of an external service like nginx.

Since our application is essentially a single page app, all we need to do is to tell whitenoise where all the files for this app are.
Whitenoise has a configuration called `WHITENOISE_ROOT`, which will help us in this case,  in our `settings.py` file lets add
```
WHITENOISE_ROOT = FLUTTER_DIR
```
And that's it, with this in place whitenoise will be able to find and serve all the files from our flutter application.

So now, if we go back and try accessing `http://localhost:8000/web` out flutter application will be visible.

