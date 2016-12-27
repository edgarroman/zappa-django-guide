# Core Django Setup

This section documents setting up a Django project with only core Python functionality responding to HTTP calls.  The value of this core walkthrough could be to power an API driven compute engine or a event-driven data processing tool without the need to provide a UI.


### Expectations and Goals

After going through this section the following will work:

* URL Routes in your Django projects
* Views can produce html / json / data output
* Management Commands

What will not work (yet - see other walkthroughs for this functionality)

* Static Files will not be served (More on that [here](static_files.md))
* There is no database connection available (not even SQLite)
* No HTTPS support

## Setup and Prerequisites 

* Python 2.7 (due to [AWS lambda only supporting 2.7](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html)) 
* Django 1.10.4
* [zappa 0.32.1](https://pypi.python.org/pypi/zappa)


## Setup AWS Account Credentials

Details in this section are light because this information is documented well elsewhere on the web.

* Create AWS Account if you haven't already
* Create an S3 bucket.  
   For purposes of this walkthrough I have used the bucket name of `zappatest-code` in the 'US Standard' region.  This bucket will be used by zappa as a mechanism to upload your project into the lambda environment.  Thus it will generally be empty except during the brief time you are deploying the project.
* Create an IAM User with API keys
   Easier said than done.  The quick and easy way of doing this is to create a user with a policy that allows a very broad set of permissions.  However, this is not great from a security perspective. There is an [ongoing discussion](https://github.com/Miserlou/Zappa/issues/244) about the exact set of permissions needed.

Now we need to allow scripts and local programs to get the credentials created above.  You have some options for this:

1. Set [environment variables](https://github.com/Miserlou/Zappa/issues/244)

    This is very easy but must be done for each bash console you are using.
   
    ```
    export AWS_ACCESS_KEY_ID=<your key here>
    export AWS_SECRET_ACCESS_KEY=<your secret access key here>
    ```
   
2. Create a local credentials file (`~/.aws/credentials` on Linux, or OS X)

	 Probably a better long term solution since you can store multiple 
	 sets of keys for different environments using profiles.

    ```
    [default]
    aws_access_key_id = your_access_key_id
    aws_secret_access_key = your_secret_access_key
    ```

### Useful links for Windows or more information:

* http://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/setup-credentials.html
* http://boto3.readthedocs.io/en/latest/guide/configuration.html#configuring-credentials

## Create local environment

It is highly recommended that you leverage virtual environments for this test project.

```
mkdir zappatest
cd zappatest
virtualenv ve
source ve/bin/activate
pip install django zappa
```

## Create very basic Django project

For the purposes of this walkthrough we are taking the most basic Django project.

```
django-admin startproject frankie .
```

### Testing the basic Django project
At this point if you run 
```
python manage.py runserver
```

And visit http://127.0.0.1:8000 with your browser you should see the standard Django 'It Worked!' page

Now quit the server using Control-C.  You should be back at the console prompt

## Setup Zappa

```
zappa init
```
You will encounter a series of prompts:

* Name of environment - just accept the default 'dev'
* S3 bucket for deployments.  Use the value of the S3 bucket you created above.  If you follow the walkthrough then use `zappatest-code`
* Zappa should automatically find the correct settings file so accept the default
* Say 'no' to deploying globally
* If everything looks ok, then accept the info

Here's a transcript of what you should see:

```
(ve) $ zappa init

███████╗ █████╗ ██████╗ ██████╗  █████╗
╚══███╔╝██╔══██╗██╔══██╗██╔══██╗██╔══██╗
  ███╔╝ ███████║██████╔╝██████╔╝███████║
 ███╔╝  ██╔══██║██╔═══╝ ██╔═══╝ ██╔══██║
███████╗██║  ██║██║     ██║     ██║  ██║
╚══════╝╚═╝  ╚═╝╚═╝     ╚═╝     ╚═╝  ╚═╝

Welcome to Zappa!

Zappa is a system for running server-less Python web applications on AWS Lambda and AWS API Gateway.
This `init` command will help you create and configure your new Zappa deployment.
Let's get started!

Your Zappa configuration can support multiple production environments, like 'dev', 'staging', and 'production'.
What do you want to call this environment (default 'dev'):

Your Zappa deployments will need to be uploaded to a private S3 bucket.
If you don't have a bucket yet, we'll create one for you too.
What do you want call your bucket? (default 'zappa-v20ssav8g'): zappatest-code

It looks like this is a Django application!
What is the module path to your projects's Django settings?
We discovered: frankie.settings
Where are your project's settings? (default 'frankie.settings'):

You can optionally deploy to all available regions in order to provide fast global service.
If you are using Zappa for the first time, you probably don't want to do this!
Would you like to deploy this application to globally? (default 'n') [y/n/(p)rimary]: n

Okay, here's your zappa_settings.js:

{
    "dev": {
        "django_settings": "frankie.settings",
        "s3_bucket": "zappatest-code"
    }
}

Does this look okay? (default 'y') [y/n]: y

Done! Now you can deploy your Zappa application by executing:

	$ zappa deploy dev

After that, you can update your application code with:

	$ zappa update dev

To learn more, check out our project page on GitHub here: https://github.com/Miserlou/Zappa
and stop by our Slack channel here: http://bit.do/zappa

Enjoy!,
 ~ Team Zappa!
(ve) $
```

### Testing the Zappa Setup

So now if we run

```
zappa deploy dev
```

But unfortunately we encounter an error: 


```
(ve) $ zappa deploy dev
Calling deploy for environment dev..
Warning! AWS Lambda may not be available in this AWS Region!
Warning! AWS API Gateway may not be available in this AWS Region!
Oh no! An error occurred! :(

==============

Traceback (most recent call last):
	[boring callback removed]
NoRegionError: You must specify a region.

==============

Need help? Found a bug? Let us know! :D
File bug reports on GitHub here: https://github.com/Miserlou/Zappa
And join our Slack channel here: https://slack.zappa.io
Love!,
 ~ Team Zappa!
(ve) $
```
Aw man, the error **NoRegionError: You must specify a region.** is holding us back.  Zappa is complaining that no AWS region is specified.  So we need to specify a region.  In this walkthrough we are leveraging `us-east-1` which corresponds to the same region we used above for the S3 bucket.

You have options:

1. Specify a default region using environment variables
 
    Again, the drawback here is this must be set for every console
   
    ```
    export AWS_DEFAULT_REGION=us-east-1
    ```
   
2. Add default region in your `~/.awd/credentials` file

	 Better but this will affect all AWS scripts and programs on your machine.
	
	 ```
	 [default]
	 aws_access_key_id = your_access_key_id
	 aws_secret_access_key = your_secret_access_key
	 region=us-east-1
	 ```

2. Edit the `zappa_settings.json` file to have an AWS region.

    Probably best option because now the zappa configuration has minimal dependencies on external user environment.
   
    ```
    {
     "dev": {
         "aws_region": "us-east-1",
         "django_settings": "frankie.settings",
         "s3_bucket": "zappatest-code"
    		} 
    }
    ```
   
    Don't forget to put commas in the proper place - JSON is fiddly!

## Deploy your project using Zappa

Now it's easy to do the initial deployment

```
zappa deploy dev
```

Zappa will automatically create an AWS API gateway that will route HTTP requests to your lambda Django project.  You should see something like:

```
(ve) $ zappa deploy dev
Calling deploy for environment dev..
Downloading and installing dependencies..
100%|███████████████████████████████████████████████████████████████████████████████████████████| 27/27 [00:07<00:00,  3.91pkg/s]
Packaging project as zip..
Uploading zappatest-dev-1482425936.zip (13.1MiB)..
100%|████████████████████████████████████████████████████████████████████████████████████████| 13.8M/13.8M [00:25<00:00, 603KB/s]
Scheduling..
Scheduled zappatest-dev-zappa-keep-warm-handler.keep_warm_callback!
Uploading zappatest-dev-template-1482425980.json (1.5KiB)..
100%|███████████████████████████████████████████████████████████████████████████████████████| 1.58K/1.58K [00:00<00:00, 2.08KB/s]
Waiting for stack zappatest-dev to create (this can take a bit)..
100%|█████████████████████████████████████████████████████████████████████████████████████████████| 4/4 [00:18<00:00,  4.69s/res]
Deploying API Gateway..
Deployment complete!: https://x6kb437rh.execute-api.us-east-1.amazonaws.com/dev
```

Brilliant!  We should be able to use a browser to visit the URL provided at the end of the script.

Once we do, however, we get:

```
DisallowedHost at /
Invalid HTTP_HOST header: 'x6kb437rh.execute-api.us-east-1.amazonaws.com'. 
You may need to add x6kb437rh.execute-api.us-east-1.amazonaws.com' to ALLOWED_HOSTS.
```

The built-in [Django security settings](https://docs.djangoproject.com/en/1.10/topics/security/#host-header-validation) are kicking in and preventing bad stuff from happening.  So we need to modify our Django settings file to accommodate the [default hostname that AWS API Gateway uses](http://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-custom-domains.html).  Note that the AWS region is part of the hostname and thus should match your selected region.

Now edit `frankie/settings.py` and change ALLOWED_HOSTS to;

```
ALLOWED_HOSTS = [ '127.0.0.1', '.execute-api.us-east-1.amazonaws.com', ]
```

Once done, we can again deploy to AWS Lambda.  But this time, since we've already pushed the initial deploy, we use the **update** action on the zappa command line.

```
zappa update dev
```

After this completes, you should be able to see your Django site in action.  Note that you will actually get a Page not found (404) response.  This indicates that your Django site is functional and working.   

