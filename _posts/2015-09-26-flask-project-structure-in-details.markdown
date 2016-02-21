---
layout: post
title:  "Flask Project Structure in Details"
date:   2015-09-26 22:18:00 +0300
tags: [flask]
---



# Introduction #

Flask documentation [describes a project structure very briefly ](http://flask.pocoo.org/docs/0.10/patterns/packages/), so this post tries to solve common problems, that faced by the new developers when they start working with real world projects.

To make it clear let's define a goals, that good project should met.



# Problem #

The good web project should solve following problems (from developers stand point)

1. **Reproducibility**. Each developer should have the same environment configuration and this environment should be extremely close to the production setup. This prevents a lot of configuration bugs;
1. **Multiple environments**. This item extends the first one. Project could be run in several different environments and should minimize possible risks about that. For instance, developer wants to install [flask-debug-toolbar ](https://flask-debugtoolbar.readthedocs.org/en/latest/)and any other package that simplifies development process, but this shouldn't go to the production or continuous integration;
1. **Simplicity**. New developers should understand project structure without much effort;
1. **Scalability**. Project could grow and structure should remains as simple as possible.
1. **Configuration is a part of the code**. Project is changing, and shipped application should reflect all changes, that have been made by the developers team.



# Solution #

Full solution is provided at [the end of this post ]({% post_url 2015-09-26-flask-project-structure-in-details %}#final-solution), but first things first.


## Generic ##

{% highlight bash %}
├── .gitignore
├── Vagrantfile
├── README.md
├── requirements.txt
├── docs
├── bin/
{% endhighlight %}

**.gitignore** is necessary for preventing commits with some junk files, such as *.pyc.

**README.md** is optional but it could contain some really basic HOWTO for developers

**requirements.txt** contains only one line in this configuration: ```-r yourapp/requirements/production.txt```. This file is used by [pip](https://en.wikipedia.org/wiki/Pip_(package_manager)) for building virtual environment and track project dependencies. I wouldn’t recommend to use ```pip freeze > requirements.txt```, because it exposes all packages installed in the current environment and also makes file really big. For instance flask package [already has a dependency](https://github.com/mitsuhiko/flask/blob/master/setup.py) for [werkzeug](http://werkzeug.pocoo.org/) and [jinja2](http://jinja.pocoo.org/), so you don’t need to specify these dependencies

Sphinx generated documentation should be placed in the **docs** folder. A lot of project ignores this, but documentation is really important and allows to significantly reduce time of knowledge transfer.

All auxiliary dev scripts could be placed in **bin/** folder. For instance, I usually add here a scripts for backup / restore database.


### Vagrant ###

**Vagrant** is very useful in development. Work flow is following:

1. [Vagrant ](https://www.vagrantup.com/) builds virtual machine and configures it, establishes shared folder with host OS and port forwarding
1. Developer edits code in that shared folder in his favorite editor or IDE
1. Developer runs the development server in VM using SSH connection

Pros:

* All developers use the same virtual machine
* Virtual machine could be provisioned from scratch
* Developer could work on any major host OS, such as Mac OS X, GNU/Linux or Windows. Furthermore, developer doesn’t need to install a lot of 3rd party software for establish the environment 
* Same setup scripts could be used in production or for building Docker images
* PyCharm professional edition supports [remote Python interpreter ](https://www.jetbrains.com/pycharm/help/configuring-remote-interpreters-via-ssh.html) and Vagrant so working process is transparent in this IDE

Cons:

* Computational overhead for Virtual Environment
* PyCharm Professional is non-free


## Configuration is a part of the code ##

{% highlight bash %}
├── deploy/
│   ├── nginx/
│   ├── gunicorn/
│   └── supervisor/
{% endhighlight %}

Tracking configuration changes with code is a good idea, because it forces developers update the configuration accordingly with the code. 

**deploy/** folder could contain any configuration files for production environment. For example ```nginx``` folder may contain ```nginx.conf``` and ```sites-available/yourapp.conf``` and it could just copied in ```etc/``` folder during deployment or building of new Docker image. 


## Flask application ##

{% highlight bash %}
├── yourapp/
│   ├── Dockerfile
│   ├── requirements/
│   │   ├── production.txt         - base requirements file
│   │   ├── develop.txt            - for developer tools, depends on production.txt [ and on ci.txt]
│   │   └── ci.txt                 - used in CI, depends only on production.txt
│   ├── templates/
│   │   ├── layouts/               - base templates used for extending
│   │   ├── macros/                - jinja macroses 
│   │   ├── errors/                - 404.html, 500.html and so on
│   │   ├── app1/                  - templates for app1
│   │   └── app2/
│   ├── static/                    - served by nginx
│   │   ├── css/                   - style sheets
│   │   ├── fonts/
│   │   ├── js/                    - javascript
│   │   ├── img/
│   │   ├── robots.txt
│   │   └── sitemap.xml
│   ├── migrations/                - database migrations
│   ├── config/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── production.py
│   │   ├── develop.py
│   │   └── ci.py
│   ├── app1/
│   │   ├── __init__.py
│   │   ├── controllers.py         - blueprint with app1 specific controllers
│   │   ├── forms.py                
│   │   └── models.py               
│   ├── app2/
│   │   └── ...
│   └── app.py                     - flask application object and manage commands interface
{% endhighlight %}

### Python requirements management ##

{% highlight bash %}
│   ├── requirements/
│   │   ├── production.txt
│   │   ├── develop.txt
│   │   └── ci.txt
{% endhighlight %}
Any python based web project uses a lot of 3rd party packages, and pip's requirements files are a good way of keep track of project dependencies. But the most progressive way, described in the "2 scoops of django" is maintain requirements not in a single file but in several with their own hierarchy. Obviously, that production.txt is the base and should contains as few dependencies as possible. develop.txt depends on production.txt and adds some useful packages for developers, such as flask-debug-toolbar, freezegun, pylint and pep8.


## Templates structure is also important ##

If your project has any web pages rendered on the server side it obviously has a templates. Maintaining a good templates structure is a important because not only developers could work with a templates, but web designers or  SEO.

* layouts/ contains base templates, that extended by all other applications. Keep it small in simple.
* macros/ contains [jinja macros](http://jinja.pocoo.org/docs/dev/templates/#macros)
* errors/ extends some base templates from layouts/ and renders error pages


## Flaks configurations ##

{% highlight bash %}
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── production.py
│   │   ├── develop.py
│   │   └── ci.py
{% endhighlight %}

Official documentation [recommends](http://flask.pocoo.org/docs/0.10/config/#configuring-from-files) to separate Flask configuration from application code, but I think it's not convenient when your application becomes large and you won't distribute it in public. 

So in this case you probably would like to have configuration under control. The main reason why each configuration is separated in different module allows to store sensitive production data in Environment Variables and raise exceptions if it's not set, and for development configuration store ```SECRET_KEY``` or other 3rd API testing keys in configuration file.

```base.py``` contains options, that remains the same for all environments, and other files corresponds to the specific environment.


# Testing #

{% highlight bash %}
└── tests
    ├── __init__.py
    ├── base.py                    - base test case and utility functions
    ├── constants.py               - testing constants
    ├── conf
    │   ├── pep8.rc
    │   └── pylint.rc
    ├── test_app1/
    │   ├── __init__.py
    │   ├── test_models.py
    │   └── test_controllers.py
    ├── test_app2/
    │   └── ... 
    └── test_integration/
{% endhighlight %}

There's several reasons to separate code of the tests and code of the application:

* Code of the tests could possible have it's own hierarchy and data fixtures
* Code of the tests doesn't go to the production
* Actual code isn't polluted with a testing python files

**pep8.rc** and **pylint.rc** could be used for static analysis of the codebase in CI environment.



# Final Solution #
{% highlight bash %}
├ project_root/
├── .gitignore                    
├── Vagrantfile
├── README.md
├── requirements.txt               - reference to the yourapp/requirement/production.txt
├── docs
├── bin/
├── deploy/
│   ├── nginx/
│   ├── gunicorn/
│   └── supervisor/
├── yourapp/
│   ├── Dockerfile
│   ├── requirements/
│   │   ├── production.txt         - base requirements file
│   │   ├── develop.txt            - for developer tools, depends on production.txt [and on ci.txt]
│   │   └── ci.txt                 - used in CI, depends only on production.txt
│   ├── templates/
│   │   ├── layouts/               - base templates used for extending
│   │   ├── macros/                - jinja macroses 
│   │   ├── errors/                - 404.html, 500.html and so on
│   │   ├── app1/                  - templates for app1
│   │   └── app2/
│   ├── static/                    - served by nginx
│   │   ├── css/                   - style sheets
│   │   ├── fonts/
│   │   ├── js/                    - javascript
│   │   ├── img/
│   │   ├── robots.txt
│   │   └── sitemap.xml
│   ├── migrations/                - database migrations
│   ├── config/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── production.py
│   │   ├── develop.py
│   │   └── ci.py
│   ├── app1/
│   │   ├── __init__.py
│   │   ├── controllers.py         - blueprint with app1 specific controllers
│   │   ├── forms.py
│   │   └── models.py
│   ├── app2/
│   │   └── ...
│   └── app.py                     - flask application object and manage commands interface
└── tests
    ├── __init__.py
    ├── base.py                    - base test case and utility functions
    ├── constants.py               - testing constants
    ├── conf
    │   ├── pep8.rc
    │   └── pylint.rc
    ├── test_app1/
    │   ├── __init__.py
    │   ├── test_models.py
    │   └── test_controllers.py
    ├── test_app2/
    │   └── ... 
    └── test_integration/
{% endhighlight %}

# Conclusion #

As a result we have:

* Vagrant improves reproducibility of the project environment
* Several requirements.txt and configuration modules allows to support multiple environments
* Flask blueprints separated into different modules allow application to scale well and stay simple
* Getting configuration of nginx and other service under version control makes a configuration to be a part of the code. 

# Links and Inspiration #

Most of the ideas have been taken from a great book ["Two scoops of django" ](http://twoscoopspress.org/collections/everything/products/two-scoops-of-django-1-8)
[Docker ](https://www.docker.com/) inspires me to maintain project simple.
Other:

* [Flask bone](https://github.com/imwilsonxu/fbone)
* [Official documentation](http://flask.pocoo.org/docs/0.10/patterns/packages/)
* [https://etscrivner.github.io/posts/2014/10/building-large-flask-apps-in-the-real-world/](https://etscrivner.github.io/posts/2014/10/building-large-flask-apps-in-the-real-world/)