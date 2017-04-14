# Setup your Environment

This section provides guidance to set up a zappa working environment.

## Why do I need a working environment?

While the ultimate goal is to have your Django application hosted in a cloud-based serverless environment, a working environment is needed to:

* Collect the required packages
* Build a lambda compatible deployment
* Upload the deployment 
* Coordinate the various AWS services to enable the cloud-based environment

In addition, a working environment assists with development and testing.  The caveat is that this working environment will not match exactly the cloud-based deployment. However, the goal is to get a reasonablly close approximation while still balancing ease of use.

## Baseline packages

To ensure baseline expectations are set, all environments will assume the following criteria:

* Python 2.7 (due to [AWS lambda only supporting 2.7](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html)) 
* Django 1.10
* Latest version of [zappa](https://pypi.python.org/pypi/zappa)

In addition, zappa *requires* a virtual environment in which to function.  So all approaches below include a virtual environment.  

### Why not Python 3?

While the Python community is rapidly moving to python 3, currently [AWS lamdba supports only python 2.7](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html)

## Approach #1 - Local Machine

You can easily set up your working environment on your local machine. For simple projects, this is very easy to manage and maintain.  All you need is Python 2.7, pip, and virtualenv installed.  This works for Windows, MacOS, and Linux machines.  

Here we setup a working environment named 'zappatest'

```sh
mkdir zappatest
cd zappatest
virtualenv ve
source ve/bin/activate
pip install django zappa
```
And you are done.  

### Cautions

While this approach is easy to get up and running, the challenge comes along when you require more advanced python packages.  

For example, once you start connecting to databases, you will need to compile packages such as 'psycopg2' for PostGresSQL.  You should consider the implications of installing needed libraries on your local machine.

## Approach #2 - Docker with zappa

Sometimes leveraging Docker to create an isolated working environment is a good idea.  It takes more work to setup initially, but once you have the foundations, it is quite easy to create multiple working environments and it is easier to share those same environments with other folks on your team.  

Most of this information is taken from the [current zappa docker project](https://github.com/danielwhatmuff/zappa).  Be sure to check that repo for current updates.

### Inital Setup 

These steps need to be performed once

* [Install Docker](https://docs.docker.com/engine/installation/)
* Pull the zappa docker image from Docker github
```sh
docker pull danielwhatmuff/zappa
```
* Create a shortcut that allows AWS credentials to pass through to the docker container
    * If you use [environment variables for AWS Credentials](aws_credentials#setup-local-account-credentials) then use:
```sh
alias zappashell='docker run -ti -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID -e AWS_DEFAULT_REGION=$AWS_DEFAULT_REGION -v $(pwd):/var/task  --rm danielwhatmuff/zappa bash'
alias zappashell >> ~/.bash_profile
```
    Be sure to define the `$AWS_DEFAULT_REGION` environment variable

    * If you use a [credentials file for AWS Credentials](aws_credentials#setup-local-account-credentials) then use:
```sh
alias zappashell='docker run -ti -e AWS_PROFILE=$AWS_PROFILE -v $(pwd):/var/task -v ~/.aws/:/root/.aws  --rm danielwhatmuff/zappa bash'
alias zappashell >> ~/.bash_profile
```
    Note that you must either define the `$AWS_PROFILE` environment variable or edit the alias above to be hardcoded to a specific profile.  Example of hardcoding the alias:
```sh
alias zappashell='docker run -ti -e AWS_PROFILE=zappa -v $(pwd):/var/task -v ~/.aws/:/root/.aws  --rm danielwhatmuff/zappa bash'
alias zappashell >> ~/.bash_profile
```

### Usage on a Project

Once the steps above are complete, then it is very easy to start a working environment.

#### First time Project setup

You will need to install any python dependencies and/or system libraries.  To fire up the docker container use:

```sh
$ cd /your_zappa_project
$ zappashell
zappa>
```

Next, create the *required* virtual environment, activate it, and install needed dependencies

```sh
zappa> virtualenv ve
zappa> source ve/bin/activate 
(ve)zappa> pip install -r requirements.txt
```

Since the virtual environment is contained in the current directory, and the current directory is mapped to your local machine, any changes you make will be persisted between Docker container instances.  

Finally, I recommend upgrading to the latest zappa project since it changes frequently
```sh
(ve)zappa> pip install --upgrade zappa
```

At this point, you are ready to start using zappa.  Once you are finished, you can simply exit the container.

#### Each time using the Project

Subsequent times you'd like to use the project, merely fire up the container:
```sh
$ cd /your_zappa_project
$ zappashell
zappa> source ve/bin/activate
(ve)zappa> 
```

### Changes to the Docker Image

Once at the zappashell prompt you can install any needed libraries.  But if you depend on libraries that are installed in the system (essentially anything out of the current directory and virtual environment), they will be lost when the container exits.

The solution is to make your own Docker Image.  Full directions of creating a Docker Image is beyond the scope of this document, but essentially there are a few steps

#### Create a Dockerfile

In the local directory create a Dockerfile (note that only the 'D' is capitalized)

```Docker
FROM danielwhatmuff/zappa

RUN yum -y install postgresql-devel

CMD ["zappa"]
```

#### Build the Docker Image

In the same directory as the Dockerfile

```sh
docker build -t myzappa .
```

#### Update your zappashell alias

To make sure it points to your new image.  Essentially replace `danielwhatmuff/zappa` with `myzappa`.  Example:
```sh
alias zappashell='docker run -ti -e AWS_PROFILE=zappa -v $(pwd):/var/task -v ~/.aws/:/root/.aws  --rm myzappa bash'
alias zappashell >> ~/.bash_profile
```


