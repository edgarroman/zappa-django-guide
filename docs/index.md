# Guide to using Django with Zappa

This repo exists to document the process of getting a standard Django project running live in AWS Lambda using the
[zappa project](https://github.com/Miserlou/Zappa). We will explore various configurations in a building-block fashion in the hopes that folks can leverage only the relevant parts for their needs.

## Setup your Environment

It is important to read this section in order to establish your working environment: [Setup your Environment](setup.md)

## Walkthroughs

### [Core Django Setup](walk_core.md)

This section documents setting up a Django project with only core Python functionality responding to HTTP calls. The value of this core walkthrough could be to power an API driven compute engine or a event-driven data processing tool without the need to provide a UI.

### [Hosting Static Files](walk_static.md)

Generally if you'd like to use your Django project to present a User Interface (UI) then you'll need to display Images and CSS and serve Javascript files. These are known as static files and to deliver them using Zappa is unlike the traditional method of hosting the static files on a Linux or Windows box. This walkthrough documents one way of hosting the files on AWS S3.

### [Using a Database](walk_database.md)

This walkthough documents the steps necessary to connect your application to a hosted database.

### [Using a Custom Domain Name](walk_domain.md)

Let's face it, the default urls provided by Zappa via API Gateway are ugly. Read this walkthrough to get a sense of what it takes to make the urls much more user friendly without a dedicated proxy service.

### [Application Adaptations](walk_app.md)

Running code in AWS Lambda is not the same as running code on a dedicated virtual server.  
This document describes differences in the AWS Lambda environment and outlines many of the possible
adaptations you may need to apply to your application.
