---
title: "Concept - Django project as a debian package"
date: "2020-03-04"
description: "packaging simple django projects as a debian package"
tags: ["debian", "django", "python"]
draft: true
---

# Concept - Django project as a debian package

For the past couple of months I've been pondering whether it would be possible to package a full django project as debian package (.deb).

Now and then I tend to write small web applications that run on my machine and use the SQLite database. On one occasion somebody asked if they could run the application on their machine.

These small web applications tend to just use Django, Django Rest Framework, JavaScript(eg: jquery) and a CSS framework that does not need to be customised or built and on some occasions a background task like celery.

One other factor is that these web applications don't get many updates. Once I've built a web application for a specific purpose it tends to stay that way for a long time.

Is there really need for docker in these cases, I don't really think so.

Why cant I just install these apps as
```
sudo apt install <web-application>
```
and leave it alone.

Lots of work has been done in this space eg:
* [Python Application Deployment with Native Packages](https://hynek.me/articles/python-app-deployment-with-native-packages/)


but I'm looking for much more narrowly scoped approach.

## What steps should be required?

* command-line to ask basic setup questions.

    ** Which django version

	** Which database (PostgreSQL/SQLite)

	** Install certain python packages (Django,whitenoise etc)
	
* Setup a virtual enviroment.

* Its should create a configuration file for development and production.

* When building it should collect all static files.

* Keep version history of web application during each build.

* configure the start/stop scripts (eg: gunicorn)

* generate a requirements file

* Turn the django project into a python package.


## What tools are already available?

The developers at spotify have really tackled the build problem very well, they have an application called [dh-virtualenv](https://github.com/spotify/dh-virtualenv) which will do all the heavy lifting for us already, this will help is in building the debian package.


The django `settings.py` file can be [configured](https://docs.djangoproject.com/en/2.2/ref/django-admin/#cmdoption-startproject-template) when starting a new project, did not know this was possible until I started the research. Django Documentation really has everything that you need. 

This will allow us to integrate a `.cfg` file for all our projects variables.


## What it wont do?

* fiddle around with default django project structure.
* change static and media files.
* build assets (js,css) and such steps.

* Its not meant to be an alternative to [cookiecutter-django](https://github.com/pydanny/cookiecutter-django). django-qoqa is very narrowly scoped for the really simple projects.


## How will it run?
 Once build the project should be be managed by systemd, should be easy as 

```
systemctl start <project_name>
```
* When a new version is deployed, It should stop the current instance of the project and install a new version.
* Once the application is running it should run as a non-root user.


## Result
I've created a small tutorial to show how this [project works](https://github.com/knightebsuku/python3-django-qoqa) 

