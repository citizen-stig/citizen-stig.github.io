---
layout: post
title:  "Using Celery with Flask Factories"
date:   2016-02-17 22:30:00 +0300
tags: [python, flask, celery, factory pattern]
comments: true
---

# Introduction #

Setting up large Flask application using factory pattern is very convinient, because it prevents a code being run at import time and provides more flexible way to setup application. 

[Celery](http://docs.celeryproject.org/en/latest/getting-started/first-steps-with-celery.html) is a good and must have tool for running asynchonious tasks, but it might be a little tricky to configure it in a large application. 


# Problem #

Imagine a following project layout

{% highlight python %}
tasks.py             # Async celery tasks, imports factories.celery
controllers.py       # Views, imports tasks
factories/
    application.py   # Applicatoin factory, imports controllers
    celery.py        # Celery factory, imports application
worker.py            # Module for worker process
{% endhighlight %}

Simple flask app is configured in `factories.application`:

{% highlight python %}
from flask import Flask

from controllers import home
from .configuration import get_config

def create_application():
    config = get_config()
    app = Flask(__name__)
    app.config.from_object(config)
    app.register_blueprint(home)
    return app 
{% endhighlight %}

In  `factories.celery` Please note, that we need to override `TaskBase`, co celery can access flask app contenxt, such as `current_app`. :

{% highlight python %}
from celery import Celery
from celery.app.task import Task as CeleryTask
from flask import Flask


def configure_celery(app: Flask) -> Celery:
    celery = Celery(app.import_name,
                    broker=application.config['CELERY_BROKER_URL'])
    celery.conf.update(application.config)
    TaskBase = celery.Task

    class ContextTask(TaskBase):
        abstract = True

        def __call__(self, *args, **kwargs):
            # Celery task will run inside app context
            with app.app_context():
                return TaskBase.__call__(self, *args, **kwargs)

    celery.Task = ContextTask
    return celery
{% endhighlight %}

`tasks.py` just contains basic task. Please note, that task binded to celery instance. `@shared_task` could've been used, but this approach is less magical:

{% highlight python %}
from factories.celery import create_celery
from factories.application import create_application
celery = create_celery(create_application())


@celery.task(name="tasks.simple_task")
def simple_task(argument):
    print(argument)
{% endhighlight %}

If controllers will import tasks.py there's a circular import.


# Solutions #

1. Import tasks inside a view function. Violates [PEP8](http://legacy.python.org/dev/peps/pep-0008/#imports)
1. Import tasks using importlib.
1. Change `factories.celery` and `tasks` to point to Celery instance placeholder, which will be configured by factory. 

Personally, I prefer a last option because it is clean and flexible and comes at little cost.

**Introducing Celery placeholder**

Let's add file, called `extensions` where celery instance will be intialized at import time:

{% highlight python %}
from celery import Celery

celery = Celery('celery_example', include=['myapp.tasks'])
{% endhighlight %}

This instance is a placeholder, it doesn't know which broker to use, but it ill be configured in factories:

{% highlight python %}
from celery import Celery
from celery.app.task import Task as CeleryTask
from flask import Flask

from myapp import extensions

def configure_celery(app: Flask) -> Celery:
    TaskBase: CeleryTask = extensions.celery.Task
    # Initialization of instance is not here anymore
    class ContextTask(TaskBase):
        abstract = True

        def __call__(self, *args, **kwargs):
            with app.app_context():
                return TaskBase.__call__(self, *args, **kwargs)
    # Configuration of placeholder happens here
    extensions.celery.conf.update(
        broker_url=app.config['CELERY_BROKER_URL'],
        # Rest of configuration
    )
    extensions.celery.Task = ContextTask
    return extensions.celery
{% endhighlight %}

`tasks.py`:
{% highlight python %}
from myapp.extensions import celery

@celery.task(name="tasks.simple_task")
def simple_task(argument):
    print(argument)
{% endhighlight %}

And in controllers:

{% highlight python %}
from flask import Blueprint, render_template, Response, request
from myapp import tasks               # Actual tasks
from myapp.extensions import celery   # For fetching results
home = Blueprint('home_views', __name__)


@home.route('/', methods=['GET', 'POST'])
def index() -> Response:
    task_id = None
    if request.method == 'POST':
        task_id = tasks.simple_task.delay(request.form.get('message'))
    return render_template('index.html', task_id=task_id)


@home.route('/result/<task_id>')
def task_result(task_id):
    result = celery.AsyncResult(task_id)
    return render_template('task_result.html', task_id=task_id, result=result)
{% endhighlight %}

And file with celery worker:

{% highlight python %}
from myapp.factories.application import create_application
from myapp.factories.celery import configure_celery

celery = configure_celery(create_application())
{% endhighlight %}

# Conclusion #

The only disadvantage of this approach is that celery instance is singleton, which is run at import time, but it should not be a problem for the most of the applications.

Working proof of concept is [available on github](https://github.com/citizen-stig/celery-with-flask-factories)
