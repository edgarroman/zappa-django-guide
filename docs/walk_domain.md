# Using a Custom Domain

If you've followed the walkthroughs thus far, you've at least created a working [Django site using Zappa](walk_core.md)
But the URL provided by Zappa is pretty darn ugly.  Not only does it use an apparent random domain name, but the Zappa environment is used as the path.  For example:

```
https://bnu0zcwezd.execute-api.us-east-1.amazonaws.com/dev/
        ^^^^^^^^^^^^^^^^^^^^^^                         ^^^
      Auto Generated API Gateway              Your Zappa Environment 
```

Ideally most sites would be something like:

```sh
https://www.zappaguide.com/
```

This is entirely possible with Zappa - so how do we get there?

## Let's talk about HTTPS

Perhaps you're wondering why we are introducting the concept of HTTPS when the topic of this walkthough is using a custom domain.  Zappa provides an automated way of creating the necessary custom domain mappings as a part of using encryption.  Thus many of the techiques described in this walkthough will ultimately end up with a custom domain along with HTTPS.  

In an effort to make the process straightforward, we are therefore bundling HTTPS as part of the walkthough.  Philosophical arguments for HTTPS are made [elsewhere](https://developers.google.com/web/fundamentals/security/encrypt-in-transit/why-https).  But with free services like "Let's Encrypt" and AWS Certificate Manner (free for API Gateways) there is no additional cost burden to leverage HTTPS certificates.

Note that we refer to HTTPS instead of SSL and or TLS where [appropriate](https://security.stackexchange.com/a/5127).

As a final note, if you are really opposed to encryption, or need unencrypted traffic for some reason, we will provide a method to accomplish this at the end of the walkthrough.

## Overview of the process

There are a number of services that are involved in this process:

 * **Domain Name Registrar** - Allows you to purchase and register domain names
 * **DNS Providers** - Allows you to host things online using that domain name
 * **Certificate Authority (CA)** - Provides encryption certifications to encrypt traffic for the site

Combined with Zappa, these services will all be used in this walkthrough.  Note that many companies and organizations can provide these services and some, like Amazon, can provide all three.

Ultimately, the AWS API Gateway will be associated with a new, dedicated CloudFront distribution that not only leverages the digital certificate to provide HTTPS, but also hides the Zappa environment path.  Finally, a DNS record will point to this new CF Distro to complete the experience for the end user.

## Registering your Custom Domain

First you need a registered domain.  It doesn't matter who your domain registrar is as long as you have control over the name server records to point to a DNS provider.  Of all the services, this one is the most generic and almost any Registar will do.

Let's choose an example domain for this walkthrough:
```
www.zappaguide.com
```

## Options Matrix

Since there are so many service providers, we focus on a couple combinations that work best.  Use the chart below to select the scenario that best matches your situation and follow only one set of instructions.

DNS Provider|CA|Notes|Instructions
------------|--|-----|------------
Route53|AWS Certificate Manager| All AWS combo makes this ridiculous easy| [see below](#route53-and-acm)
Route53|Let's Encrypt| Another good option that Zappa has smoothed the way| see below
Other DNS|ACM or Let's Encrypt| Some minor additional steps| see below
Other DNS|Other| You got some work to do| see below

## Route53 and ACM

### Step 1: Create a Hosted Zone in Route53

If your Registrar is also Route53, skip this step and move on to Step 2.  AWS did this for you when you registered the domain.

Follow the instructions for [creating a hosted zone in Route53](aws_route53.md#create-a-hosted-zone-in-route53)

### Step 2: Create your digital certificate in ACM

Follow the instructions for [requesting a certificate in the ACM console](aws_acm.md##request-a-certificate)

Be sure to record the ARN for the newly issued certificate.

### Step 3: Edit the Zappa Settings File

Now we add the following to our Zappa settings file.  These settings prepare Zappa to configure our API gateway properly.

``` hl_lines="10 11"
{
    "dev": {
        "django_settings": "frankie.settings", 
        "s3_bucket": "zappatest-code",
        "aws_region": "us-east-1",
        "vpc_config" : {
            "SubnetIds": [ "subnet-f3446aba","subnet-c5b8c79e" ], // use the private subnet
            "SecurityGroupIds": [ "sg-9a9a1dfc" ]
        },
        "certificate_arn": "arn:aws:acm:us-east-1:738356466015:certificate/1d066282-ce94-4ad7-a802-2ff87d32b104",
        "domain": "www.zappaguide.com",
    }
}
```

For the `certificate_arn` use the ARN value obtained in step 2 above.

### Step 4: Run Certify

This final step triggers your local Zappa environment to reach out to AWS and configure your API Gateway to honor the domain name specified.

```
(ve) $ zappa certify dev
Calling certify for environment dev..
Are you sure you want to certify? [y/n] y
Certifying domain www.zappaguide.com..
Created a new domain name with supplied certificate. Please note that it can take up to 40 minutes for this domain to be created and propagated through AWS, but it requires no further work on your part.
Certificate updated!
(ve) $ 
```

And that should work fine going forward

!!! Warning
    This command must be run in the US East (N. Virginia) (us-east-1).  See [AWS documentation](http://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-custom-domains.html#how-to-custom-domains-prerequisites) for more details.

## Route53 and Let's Encrypt

Under Construction

## Other DNS and ACM or Let's Encrypt

Under Construction 

## Other Service Providers

Under Construction

