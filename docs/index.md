# Zappa Django example

This repo exists to document the process of getting a standard Django project running live in AWS Lambda using the 
[zappa project](https://github.com/Miserlou/Zappa)

## Setup and Prerequisites 

* Python 2.7 (due to [AWS lambda only supporting 2.7](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html)) 
* Django 1.10.4
* [zappa 0.32.1](https://pypi.python.org/pypi/zappa)

## Walkthroughs

### [Core Django Setup](core_django_setup.md)

This section documents setting up a Django project with only core Python functionality responding to HTTP calls.  After this section the following will work:

* URL Routes
* Views (but no static file serving)
* Management Commands

What will not work:

* Static Files not being served
* There is no database connection available (not even SQLite)
* No HTTPS support

### [Support for Static Files](static_files.md)

Now how to get static files hosted