---
layout: post
title:  "Using Celery with Flask Factories"
date:   2016-02-17 22:30:00 +0300
tags: [flask, celery, factory pattern]
---

# Introduction #

Setting up large Flask application using factory pattern is very convinient, because it prevents a code being run at import time and provides more flexible way to setup application. 

Celery is a good and must have tool for running asynchonious tasks, but it's a little bit tricky to configure it in a large application. 


# Problem #

Imagine a following project layout

{% highlight python %}
tasks.py             # Celery tasks import factories.celery
controllers.py       # Blueprints. Here we what to run some celery tasks
factories/
    application.py   # Applicatoin factory, imports controllers.py
    celery.py        # Celery factory, imports application.py
{% endhighlight %}

 Application factory:

{% highlight python %}
from flask import Flask
from .configuration import get_config
from controllers import home

def create_application():
    config = get_config()
    app = Flask(__name__)
    app.config.from_object(config)
    app.register_blueprint(home)
    return app 
{% endhighlight %}

Celery factory:

{% highlight python %}
from celery import Celery
from factories.application import create_application

def create_celery(application=None):
    application = application or create_application
    celery = Celery(application.import_name,
                    broker=application.config['CELERY_BROKER_URL'])
    celery.conf.update(application.config)
    TaskBase = celery.Task

    class ContextTask(TaskBase):
        abstract = True

        def __call__(self, *args, **kwargs):
            with application.app_context():
                return TaskBase.__call__(self, *args, **kwargs)

    celery.Task = ContextTask
    return celery
{% endhighlight %}

Tasks:

{% highlight python %}
# -*- encoding: utf-8 -*-
from factories.celery import create_celery
from factories.application import create_application
celery = create_celery(create_application())


@celery.task(name="tasks.simple_task")
def simple_task(argument):
    print(argument)
{% endhighlight %}

So if controllers will import tasks.py there's a circular import.


# Solutions #

1. Import tasks inside a view function. Violates [PEP8](http://legacy.python.org/dev/peps/pep-0008/#imports)
1. Import tasks using importlib.
1. Change celery_factory and don't import tasks and run [them by name](http://docs.celeryproject.org/en/latest/faq.html#can-i-call-a-task-by-name)

Personally, I prefer a last solution, because it gives more flexibility for structuring codebase of the tasks.

**Change celery factory**

To make celery factory more reusable let's remove dependency of application factory and make application parameter mandatory:

{% highlight python %}
from celery import Celery

def create_celery(application):
    celery = Celery(application.import_name,
                    broker=application.config['CELERY_BROKER_URL'])
    celery.conf.update(application.config)
  	# ...
{% endhighlight %}

And in controllers:

{% highlight python %}
from flask import Blueprint, current_app
from factories.celery import create_celery
home = Blueprint('home_views', __name__)

@home.route('/')
def index():
    celery = create_celery(current_app)
    res = celery.send_task('tasks.simple_task', args=('-=-= TEST FROM VIEW =-=-',))
{% endhighlight %}

And file with celery worker:

{% highlight python %}
from factories.celery import create_celery
from factories.application import create_application

celery = create_celery(create_application())
{% endhighlight %}

# Conclusion #

Calling celery tasks by name looks clearer and less magical, so I 

Working proof of concept [available on github](https://github.com/citizen-stig/celery-with-flask-factories)
