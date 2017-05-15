# Adapting your Application to Zappa / Lambda

If you've done the walkthroughs thus far, they have allowed you to seamlessly get your existing application
(or allowed you to create a new one) in AWS Lambda using [Zappa](https://github.com/Miserlou/Zappa).  

But running code in AWS Lambda is not the same as running code on a dedicated virtual server.  
This document describes differences in the AWS Lambda environment and outlines many of the possible
adaptations you may need to apply to your Application.

## Minimal disk storage might be persistent

When your code runs, whether it was trigged from an HTTP request or other event, the only disk storage
available to write files is in `/tmp`.  And that storage is limited to 500MB.  Always check the [AWS Lambda Limits](http://docs.aws.amazon.com/lambda/latest/dg/limits.html#limits-list) page because this limitation could change over time.

Interestingly, once your code runs and is complete, the AWS Lamda service 'freezes' your container and if trigged again, [could 'unfreeze' the container for reuse](http://docs.aws.amazon.com/lambda/latest/dg/lambda-introduction.html).  What this means from a
practical standpoint is that if you need to write out files to `/tmp` your code could both find files
from previous runs and also run out of disk space.

The obivous adaptations when writing out files:

* Do not rely on having files exist between code invocations
* If the temp files are of significant size, it would be better to clean them up on exit to avoid future code invocations from running out of space
* Any content uploaded via HTTP (or downloaded/created during invocation) must be persisted elsewhere such as in S3
* If you need unique files on disk for each invocation, be sure the space required per invocation
multipied by the number of possible invocations is less than 500MB.

Some additional use cases:

### Poor Man's Search Engline
You could use the temp space to power file-based tools such as the [Whoosh Search Engine](http://whoosh.readthedocs.io/en/latest/) by downloading the search index from S3.  This may work for small indexes.

## Passing Environment Variables to your Application

There are a number of ways to pass information to your application

### Environment Variables in Zappa Settings

You can include variables in the zappa settings file directly.  These variables are easy to set and are included in each deployment.  Great for variables that do not change often or are not sensitive (e.g. credentials).

```
{
    "dev": {
        ...
        "environment_variables": {
            "some_key": "some_value"
        }
    },
    ...
}
```

And then you can easily retrieve the information from within your code:

```py
import os
some_value = os.environ.get('some_key')
```

### Lambda Environment Variables

Your code can pull information from the execution environment by using the built-in AWS Lambda environment variables.  There is also a method for adding custom variables via the AWS Console.
This method is generally only useful for system-generated variables since custom variables can more easily be configured in zappa settings (see above).

First, there are the [standard environment variables](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html) such as path to code, region, and python path.  These values are automatically calculated and driven by AWS.  

In addition to these system variables, you can set [custom environment variables in the AWS Console](http://docs.aws.amazon.com/lambda/latest/dg/env_variables.html).  
So you could add `SOME_LAMBDA_KEY` in the AWS console and retrieve it in your code:

```py
import os
some_lamda_key = os.environ.get('SOME_LAMBDA_KEY')
# or get system values
aws_lambda_function_name = os.environ.get('AWS_LAMBDA_FUNCTION_NAME')
```

While on the topic of system-generated information, your code can also pull important information from 
the [Python execution context](http://docs.aws.amazon.com/lambda/latest/dg/python-context-object.html):

> While a Lambda function is executing, it can interact with the AWS Lambda service to get useful runtime information such as:

> * How much time is remaining before AWS Lambda terminates your Lambda function (timeout is one of the Lambda function configuration properties).
  * The CloudWatch log group and log stream associated with the Lambda function that is executing.
  * The AWS request ID returned to the client that invoked the Lambda function. You can use the request ID for any follow up inquiry with AWS support.
  * If the Lambda function is invoked through AWS Mobile SDK, you can learn more about the mobile application calling the Lambda function.

