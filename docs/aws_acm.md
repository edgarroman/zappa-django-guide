# AWS Credential Manager (ACM)

This page lists various activites that may be necessary to perform when leveraging Zappa

## Request a Certificate

ACM provides digital certificates for free but the certificates can only be used with [Elasic Load Balancing and Amazon CloudFront](https://docs.aws.amazon.com/console/acm/supported-services).

 1. Navigate to the [ACM Console](https://console.aws.amazon.com/acm/) and click `Request a Certificate`
 2. In 'Add a Domain name' enter 
 ![Step 1: Add a Domain name](images/aws_amc_request.png)
 Note that we entered both the 'www' subdomain and the *apex* of the domain.  This allows users to leverage either url and have it covered until a single certificate.

    Carefully consider which domains shall be covered by this certificate because once it is validated, you cannot modify the list of domains; any changes will require a new certificate to be issued.

 3. Click on Review and Request

    You should see a message similar to the image below:
    ![Step 3: Confirm](images/aws_acm_request3.png)

    Notice that ACM's perferred method of domain ownership validate is to send an email to the registered contact address in the WHOIS for the domain.  In addtion, a few select email addresses are also included.  Full [validation rules](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-validate.html) are posted in ACM Documentation.

 4. Check your email for validation links

    You should receive at least one email for each domain you entered.  You may actually get multiple email addresses because sometimes registered emails are duplicated for Techincal or Administrative contacts in the WHOIS information.  The emails should be similar to:
    ![ACM Validation Email](images/aws_acm_validate_email.png)

 5. Click on all the validation links

    **You must click on the validation link for every domain name included in step 2 above**.  The digital certificate will not be issued until all domains have been verified.
    ![ACM Validation 1](images/aws_acm_validate1.png)
    ![ACM Validation 2](images/aws_acm_validate2.png)

 6. Record the ARN for the digital certificate

    You will use the ARN for other purposes.  The ARN is displayed on the verification page but also in the details page for the certificate in the ACM console.