# Using a Custom Domain

If you've followed the walkthroughs thus far, you've at least created a working [Django site using Zappa](walk_core.md)
But the URL provided by Zappa is pretty darn ugly.  Not only does it use an apparent random domain name, but the Zappa environment is used as the path.  For example:

```sh
https://bnu0zcwezd.execute-api.us-east-1.amazonaws.com/dev/
        ^^^^^^^^^^^^^^^^^^^^^^                         ^^^
      Auto Generated API Gateway              Your Zappa Environment
```

Ideally most sites would be something like:

```sh
https://www.contoso.com/
```

This is entirely possible with Zappa - so how do we get there?

Notes here:

# Domain Name Prep

## Hosting elsewhere

## Hosting on Route53

 * Create hosted zone

# Get SSL Cert

## outside

- import the cert into acm
- but you have to deal with renewals yourself
https://aws.amazon.com/certificate-manager/faqs/

## ACM
do this from the east region and it will distribute
https://aws.amazon.com/certificate-manager/faqs/

- add domain
- the look for email
- verify and get the ARN

good for 13 months
https://aws.amazon.com/certificate-manager/faqs/

# put in zappa settings

        "certificate_arn": "arn:aws:acm:us-east-1:738356466015:certificate/1d066282-ce94-4ad7-a802-2ff87d32b104",
        "domain": "www.contoso.com",

# Run 

```
zappa certify dev
```

so what happens is that zappa requests that the api gateway associate the custom domain and ssl to a new, hidden cloudfront distribution.  It's hard to figure out what the settings are on this cf distro so just roll with it.
if you are a control freak, just make your own

it takes some time, but now hit it

https://www.contoso.com




http://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-custom-domains.html




