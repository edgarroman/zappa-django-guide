# Static Files Setup

Generally if you'd like to use your Django project to present a User Interface (UI) then you'll need to display Images and CSS and serve Javascript files.  These are known as [static files](https://docs.djangoproject.com/en/1.10/howto/static-files/deployment/) and to deliver them using Zappa is unlike the traditional method of hosting the static files on a Linux or Windows box.  

### Static files and Code on a Single Server

A very common configuration you may see recommended is to have your Django project [deployed on a server with your static files](https://docs.djangoproject.com/en/1.10/howto/static-files/deployment/#serving-the-site-and-your-static-files-from-the-same-server).  Then the advice is to have your web server software (apache, nginx, or other) have special mechanisms to directly serve the static files.  The idea is to have the fast web server software handle delivering the static images to clients and the comparatively slow Django/python code process the more complex views and page content.  

Because Zappa runs in the serverless lambda environment, this approach is not feasible since you cannot configure the web server to handle various url paths differently.  Thus another approach must be taken.

### Leveraging WSGI app to serve files

The situation where one does not have access to the web server software configuration is more common than one may think.  Hosting in a shared environment, or on Platform as a Service (PaaS) like OpenShift may prevent full configuration of the web server to effectively serve static files.  

There are ways to leverage the WSGI application (Django for us) and instruct it to serve static files.  Normally, Django treats URL requests as an opportunity to run python code.  And the python code may have complex logic.  But there is a model called [WhiteNoise] (https://github.com/evansd/whitenoise).  It is an app that will minimize the python code processing to more efficiently serve static files.  Thus no external web server software configuration is required.  While perhaps not as optimal as having the web server hosting the files, this method has been used in production effectively.  

### Using external services to serve files

Finally, there is an option to use an [external service to serve static files](https://docs.djangoproject.com/en/1.10/howto/static-files/deployment/#serving-static-files-from-a-cloud-service-or-cdn).  This is the option that is the subject of this walkthrough.

While any external service that serves files over HTTP could work, the focus for us will be to leverage the AWS service of S3 and the Content Delivery Network (CDN) of CloudFront to meet our needs.  

The S3 service will contain our files and provide the fundamental HTTP/HTTPS service.  This alone will suffice for many recreational projects, but more professional project will want to leverage CloudFront to provide caching, faster delivery, and better protection of assets.

### Using a CDN for the entire project

There are also advantages to serving the entire Django project (Lambda functions and S3 Static files) via the CloudFront CDN.  This option will not be covered in this Walkthrough.


## Setup and Prerequisites 

* Python 2.7 (due to [AWS lambda only supporting 2.7](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html)) 
* Django 1.10.4
* [zappa 0.32.1](https://pypi.python.org/pypi/zappa)

## Walkthrough

