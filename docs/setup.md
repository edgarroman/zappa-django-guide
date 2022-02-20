# Setup your Environment

This section provides guidance to set up a zappa working environment.

## Why do I need a working environment?

While the ultimate goal is to have your Django application hosted in a cloud-based serverless environment, a working environment is needed to:

-   Collect the required packages
-   Build a lambda compatible deployment
-   Upload the deployment
-   Coordinate the various AWS services to enable the cloud-based environment

In addition, a working environment assists with development and testing. The caveat is that this working environment will not match exactly the cloud-based deployment. However, the goal is to get a reasonablly close approximation while still balancing ease of use.

## Baseline packages

To ensure baseline expectations are set, all environments will assume the following criteria:

-   Python 3 (according to [AWS lambda support](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html))
-   Django 3.2 or newer
-   Latest version of [zappa](https://pypi.python.org/pypi/zappa)

In addition, zappa _requires_ a virtual environment in which to function. So all approaches below include a virtual environment.

## Approach #1 - Local Machine

You can easily set up your working environment on your local machine. For simple projects, this is very easy to manage and maintain. All you need is Python 3, pip, and virtualenv installed. This works for Windows, MacOS, and Linux machines.

Here we setup a working environment named 'zappatest'

```sh
mkdir zappatest
cd zappatest
virtualenv ve
source ve/bin/activate
pip install django zappa
```

And you are done.

!!! Warning

    While this approach is easy to get up and running, the challenge comes along when you require more advanced python packages.

    For example, once you start connecting to databases, you will need to compile packages such as 'psycopg2' for PostGresSQL.  You should consider the implications of installing needed libraries on your local machine.

    This is approach is not recommended for any type of serious zappa effort.

## Approach #2 - Docker with zappa (recommended)

Sometimes leveraging Docker to create an isolated working environment is a good idea. It takes more work to setup initially, but once you have the foundations, it is quite easy to create multiple working environments and it is easier to share those same environments with other folks on your team.

The main goal of using Docker is to create an environment that closely matches the AWS lambda environment. The closer it matches, then there will be less difficult-to-debug problems.

We will leverage the work others have done to enable such an environment. First and foremost, the folks from [lambci](https://github.com/lambci/lambci) have created github repo called [docker-lambda](https://github.com/lambci/docker-lambda) that accurately reflects the lambda environment. It provides:

-   Multiple uses
    -   A 'build' image for compilation, package creation, and deployment
    -   A 'run' image for testing and execution of your code

For the purposes of this walkthrough we will focus only on the 'build' image that provides a very nice interactive working environment for zappa. Further research into how to use the 'run' image is left as an exercise for the reader.

-   Multiple Python version support
    -   Python 3.6  (x86) ([lambci/lambda:build-python3.6](https://hub.docker.com/r/lambci/lambda/tags/))
    -   Python 3.7 (x86) ([lambci/lambda:build-python3.7](https://hub.docker.com/r/lambci/lambda/tags/))
    -   Python 3.8 (x86) ([lambci/lambda:build-python3.8](https://hub.docker.com/r/lambci/lambda/tags/))
    -   Python 3.8 (ARM & x86) ([mlupin/docker-lambda:python3.8-build](https://hub.docker.com/r/mlupin/docker-lambda/tags))
    -   Python 3.9 (ARM & x86) ([mlupin/docker-lambda:python3.9-build](https://hub.docker.com/r/mlupin/docker-lambda/tags))

Note that this work was originally inspired from [danielwhatmuff/zappa](https://github.com/danielwhatmuff/zappa) but has been enhanced to illustrate support for Python 3

### Inital Setup

These steps need to be performed once for a new system

-   [Install Docker](https://docs.docker.com/engine/installation/)
-   Pull the zappa docker image from Docker github

    ```sh
    # For Python 3.6 projects
    docker pull lambci/lambda:build-python3.8
    ```

-   Create a shortcut that allows AWS credentials to pass through to the docker container

    -   If you use [environment variables for AWS Credentials](aws_credentials#setup-local-account-credentials) then use:

    ```sh
    alias zappashell='docker run -ti -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID -e AWS_DEFAULT_REGION=$AWS_DEFAULT_REGION -v "$(pwd):/var/task"  --rm lambci/lambda:build-python3.6 bash'
    alias zappashell >> ~/.bash_profile
    ```

    Be sure to define the `$AWS_DEFAULT_REGION` environment variable

    -   If you use a [credentials file for AWS Credentials](aws_credentials#setup-local-account-credentials) then use:

    ```sh
    alias zappashell='docker run -ti -e AWS_PROFILE=$AWS_PROFILE -v "$(pwd):/var/task" -v ~/.aws/:/root/.aws  --rm lambci/lambda:build-python3.6 bash'
    alias zappashell >> ~/.bash_profile
    ```

    Note that you must either define the `$AWS_PROFILE` environment variable or edit the alias above to be hardcoded to a specific profile. Example of hardcoding the alias:

    ```sh
    alias zappashell='docker run -ti -e AWS_PROFILE=zappa -v "$(pwd):/var/task" -v ~/.aws/:/root/.aws  --rm lambci/lambda:build-python3.6 bash'
    ```

### Taking a test drive

So let's try this out now. Examples going forward will focus on Python 3.6. To fire up the docker container use:

```sh
$ cd /your_zappa_project
$ zappashell
bash-4.2#
```

Next, create the _required_ virtual environment, activate it, and install needed dependencies

```sh
bash-4.2# virtualenv ve
bash-4.2# source ve/bin/activate
(ve) bash-4.2# pip install -r requirements.txt
```

Since the virtual environment is contained in the current directory, and the current directory is mapped to your local machine, any changes you make will be persisted between Docker container instances. But if you depend on libraries that are installed in the system (essentially anything out of the current directory and virtual environment), they will be lost when the container exits. The solution for this is to create a custom Dockerfile (see below)

Finally, I recommend upgrading to the latest zappa project since it changes frequently

```sh
(ve) bash-4.2# pip install --upgrade zappa
```

!!! Warning

    It is very important that you install and activate the virtualenv only in the docker shell. This will prevent any incompatibilities with the local system environment and the docker environment.

At this point, you are ready to start using zappa. Once you are finished, you can simply exit the container.

### Project Setup

Once the steps above are complete, then it is very easy to start a working environment. But generally additional steps are required for package compilation and customizations.

#### Create a Dockerfile

Create a local Dockerfile for your project so you can easily modify needed libraries. Generally this can go in the root of your zappa project.

```Docker
FROM lambci/lambda:build-python3.6

LABEL maintainer="<your@email.com>"

WORKDIR /var/task

# Fancy prompt to remind you are in zappashell
RUN echo 'export PS1="\[\e[36m\]zappashell>\[\e[m\] "' >> /root/.bashrc

# Create and Activate the virtual environment 
RUN echo 'virtualenv -p python3 ./ve >/dev/null' >> /root/.bashrc
RUN echo 'source ./ve/bin/activate >/dev/null' >> /root/.bashrc

# Additional RUN commands here
# RUN yum clean all && \
#    yum -y install <stuff>

CMD ["bash"]
```

#### Build the docker image

```sh
$ cd /your_zappa_project
$ docker build -t myzappa .
```

This will create a local Docker image on your system.

#### Update your zappashell alias

To make sure it points to your new image. Essentially replace `lambci/lambda:build-python3.6` with `myzappa`. Example:

```sh
alias zappashell='docker run -ti -e AWS_PROFILE=zappa -v "$(pwd):/var/task" -v ~/.aws/:/root/.aws  --rm myzappa'
alias zappashell >> ~/.bash_profile
```

### Using your environment

Each time you are working on your project, merely fire up the container:

```sh
$ cd /your_zappa_project
$ zappashell
(ve) zappashell>
```
Since the virtual environment is contained in the current directory, and the current directory is mapped to your local machine, any changes you make will be persisted between Docker container instances. But if you depend on libraries that are installed in the system (essentially anything out of the current directory and virtual environment), they will be lost when the container exits. The solution for this is to add these installations as RUN commands in the Dockerfile.  

All zappa commands can be used to deploy your project:

```sh
(ve) zappashell> zappa status dev
```

#### ZappaDock
There is a simple tool that automates the above proccess called [ZappaDock](https://github.com/dickermoshe/zappadock). It make all of the above work with just one command. 
