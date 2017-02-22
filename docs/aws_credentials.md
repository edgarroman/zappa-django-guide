# Managing AWS Credentials

## Getting Started with AWS and Zappa

Details in this section are light because this information is documented well elsewhere on the web.

* Create AWS Account if you haven't already
* Create an S3 bucket.  
   For purposes of this walkthrough I have used the bucket name of `zappatest-code` in the 'US Standard' region.  This bucket will be used by zappa as a mechanism to upload your project into the lambda environment.  Thus it will generally be empty except during the brief time you are deploying the project.
* Create an IAM User with API keys
   Easier said than done.  The quick and easy way of doing this is to create a user with a policy that allows a very broad set of permissions.  However, this is not great from a security perspective. There is an [ongoing discussion](https://github.com/Miserlou/Zappa/issues/244) about the exact set of permissions needed.

Now we need to allow scripts and local programs to get the credentials created above.  You have some options for this:

## Setup Local Account Credentials

1. Set [environment variables](https://github.com/Miserlou/Zappa/issues/244)

    This is very easy but must be done for each bash console you are using.
   
    ```
    export AWS_ACCESS_KEY_ID=<your key here>
    export AWS_SECRET_ACCESS_KEY=<your secret access key here>
    ```
   
2. Create a local credentials file (`~/.aws/credentials` on Linux, or OS X)

	 Probably a better long term solution since you can store multiple 
	 sets of keys for different environments using profiles.  In addition, you can provide multiple profiles that provides some isolation between AWS accounts and/or roles.  The alternate profile example shown below is called 'zappa'

    ```
    [default]
    aws_access_key_id = your_access_key_id
    aws_secret_access_key = your_secret_access_key

    [zappa]
    aws_access_key_id = your_access_key_id_specific_to_zappa
    aws_secret_access_key = your_secret_access_key_specific_to_zappa
    ```

    Since you have multiple profiles, it is recommended that you use an environment variable to distinguish which profile is desired to be active.  Shown here is an example of using the 'zappa' profile:

    ```sh
    export AWS_PROFILE=zappa
    ```

### Useful links for Windows or more information:

* http://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/setup-credentials.html
* http://boto3.readthedocs.io/en/latest/guide/configuration.html#configuring-credentials
