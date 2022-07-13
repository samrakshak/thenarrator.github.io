## [AWS Storage Blog](https://aws.amazon.com/blogs/storage/)

# Enable password authentication for AWS Transfer Family using AWS Secrets Manager (updated)

by Warren Paull | on 05 NOV 2020 | in [AWS Lambda](https://aws.amazon.com/blogs/storage/category/compute/aws-lambda/ "View all posts in AWS Lambda"), [AWS Secrets Manager](https://aws.amazon.com/blogs/storage/category/security-identity-compliance/aws-secrets-manager/ "View all posts in AWS Secrets Manager"), [AWS Serverless Application Model](https://aws.amazon.com/blogs/storage/category/compute/aws-serverless-application-model/ "View all posts in AWS Serverless Application Model"), [AWS Transfer Family](https://aws.amazon.com/blogs/storage/category/migration/aws-transfer-family/ "View all posts in AWS Transfer Family"), [Compute](https://aws.amazon.com/blogs/storage/category/compute/ "View all posts in Compute"), [Migration & Transfer Services](https://aws.amazon.com/blogs/storage/category/migration/ "View all posts in Migration & Transfer Services"), [Security, Identity, & Compliance](https://aws.amazon.com/blogs/storage/category/security-identity-compliance/ "View all posts in Security, Identity, & Compliance"), [Storage](https://aws.amazon.com/blogs/storage/category/storage/ "View all posts in Storage"), [Technical How-To](https://aws.amazon.com/blogs/storage/category/post-types/technical-how-to/ "View all posts in Technical How-to") | [Permalink](https://aws.amazon.com/blogs/storage/enable-password-authentication-for-aws-transfer-family-using-aws-secrets-manager-updated/) |  [Comments](https://commenting.awsblogs.com/embed.html?disqus_shortname=aws-storage-blog&disqus_identifier=4674&disqus_title=Enable+password+authentication+for+AWS+Transfer+Family+using+AWS+Secrets+Manager+%28updated%29&disqus_url=https://aws.amazon.com/blogs/storage/enable-password-authentication-for-aws-transfer-family-using-aws-secrets-manager-updated/) |  [Share](https://aws.amazon.com/blogs/storage/enable-password-authentication-for-aws-transfer-family-using-aws-secrets-manager-updated/#)

[AWS Transfer Family](https://aws.amazon.com/aws-transfer-family/) provides a service-managed directory to store user credentials for users authenticating with an SSH key over the Secure File Transfer Protocol (SFTP). If you must authenticate users by password, connect using the older File Transfer Protocol (FTP) and File Transfer Protocol Secure (FTPS), or would just like to integrate with your own user directory, the service supports a custom authentication provider setup via API Gateway.

It has been just over a year since I published the [original version](https://aws.amazon.com/blogs/storage/enable-password-authentication-for-aws-transfer-for-sftp-using-aws-secrets-manager/) of this blog. Back then, I covered how you could use [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) as a secure location to store user credentials that you can access via API. Doing so enables you to use AWS Secrets manager as a custom authentication provider for password-based authentication.

Using the custom authentication mode gives you the flexibility to check other user attributes. In this post, I cover integrating AWS Secrets Manager as a custom authentication provider for AWS Transfer Family, enabling:

-   SSH and/or password-based user authentication
-   Independent password and role support for FTP/FTPS users
-   Optional client source IP address checks for individual users
-   Dynamic role allocation for access to Amazon S3
-   [Logical directory](https://aws.amazon.com/about-aws/whats-new/2019/09/aws-transfer-for-sftp-now-supports-logical-directories-for-amazon-s3/) mappings

The template uses the [AWS Serverless Application Model (AWS SAM)](https://aws.amazon.com/serverless/sam/) format. To deploy, first make sure you have the [AWS SAM CLI tools installed](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html). Then, download and unzip [this package](https://s3.amazonaws.com/aws-transfer-resources/custom-idp-templates/aws-transfer-custom-idp-secrets-manager-sourceip-protocol-support-apig.zip). Change directory into the unzipped package and run: `sam deploy –guided`.

Note: The updated package uses the format “serverid/username” when creating users in AWS Secrets Manager. This is different to the previous version where they followed the format “SFTP/username.” In addition, it is common practice to use separate authentication and authorization for FTP users due to its unencrypted communications.

## Custom authentication identity providers

By default, a new AWS Transfer Family endpoint uses the service managed, internal user directory for SSH key-based authentication, but you can instead use an IdP of your choice. To do this, specify `--identity-provider-type API_GATEWAY` with an API Gateway endpoint to provide integration to the custom authentication provider.

With this configuration, the following workflow authenticates and authorizes users:

![With this AWS Transfer Family configuration, the diagramed workflow authenticates and authorises users.](https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2020/11/05/With-this-AWS-Transfer-Family-configuration-the-diagramed-workflow-authenticates-and-authorises-users..jpg.png)

The steps are as follows:

1.  A user attempts to log in, supplying either a user name and password (SFTP, FTPS or FTP), or a user name and locally stored private SSH key (SFTP).
2.  AWS Transfer Family passes these credentials along with the client protocol and source IP address to the API Gateway endpoint you provide when creating the AWS Transfer Family endpoint. If the user does not provide a password, it is assumed that they are using SSH key-based authentication. The API Gateway integrates with an [AWS Lambda function](https://docs.aws.amazon.com/apigateway/latest/developerguide/getting-started-with-lambda-integration.html).
3.  The AWS Lambda function queries the custom authentication provider (which can be any datastore, and in this case AWS Secrets Manager) passing the same parameters from step 2.
4.  Secrets Manager returns the key-value pairs associated with the user or secret. This contains the user’s stored password, the IAM role mapping for the user, and any public SSH key information (if you allow SSH key-based authentication for the user). It also contains source IP CIDRS for you to check, and any virtual directory mappings.
5.  The AWS Lambda function validates the login and returns any user configuration. For more details on what our AWS Lambda function is going to do, see the “AWS Lambda function” section.

The API Gateway endpoint provides a single method with a resource path:

/servers/serverId/users/username/config

The `serverId` and `username` values come from the RESTful resource path. For security, the workflow sends the user’s password in the password header of the request. In addition, the workflow sends the client source IP address and protocol as part of the query string.

The payload returned in the API Gateway method response consists of the following values:

-   **Role**: ARN of the IAM Role that contains the policy used to provide the user access to your S3 bucket.
-   (Optional) **HomeDirectory**: The S3 bucket prefix that the user will be dropped into when they log in.
-   (Optional) **Policy**: An [STS.AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) scope-down policy blob that further restricts the IAM role based on certain parameters.
-   (Optional) **PublicKeys**: If no password was provided (SSH key-based authentication), then the public SSH key associated with the user is returned. This is a comma-separated list and can contain two public keys.
-   (Optional) **HomeDirectoryDetails**: This defines a logical bucket structure (symlink) or can be used to restrict the user to their home directory (chroot). [See this blog on simplifying your AWS SFTP structure with chroot and logical directories](https://aws.amazon.com/blogs/storage/simplify-your-aws-sftp-structure-with-chroot-and-logical-directories/) for more information.

This template provided uses AWS Secrets Manager as a secure data store. This enables you to create user names associated with an AWS Transfer Family server, and store the user’s custom attributes (password, IAM role, etc.).

You cannot directly connect AWS Transfer Family to Secrets Manager today, so AWS Lambda provides the logic to connect them. This AWS Lambda function is responsible for validating the user credentials against the one stored, and for returning access information.

## Get me up and running!

This package launches an AWS Transfer Family endpoint, along with an Amazon API Gateway endpoint and associated Lambda function to integrate with AWS Secrets Manager as a custom authentication provider.

As you go through the guided deployment, prompts are provided for the following information.

-   `Stack Name` – Provide a friendly name.
-   `AWS Region` – The AWS Region identifier to deploy the stack into.
-   `Create Server` – Do you want to also deploy the AWS Transfer Family Server, or just deploy the integration stack? Valid answers are true or false.
-   `SecretsManagerRegion` – If using a different Region for Secrets Manager, provide it here, otherwise, leave empty.
-   `TransferEndpointType` – Valid answers are PUBLIC or VPC.
-   `TransferSubnetIDs` – Specify the VPC subnets to deploy endpoints. The subnet IDs are comma separated with no spaces. Only needed if using a VPC deployment.
-   `TransferVPCID` – The ID of the VPC to be deployed into. Only needed if using a VPC deployment.

Once you have deployed this, you can start creating your users in AWS Secrets Manager. _Note that this revised template creates users that are unique to a specific AWS Transfer Family Endpoint (Server ID)._

1.  In the [AWS Secrets Manager console](https://console.aws.amazon.com/secretsmanager), create a new secret by choosing **Store a new secret**.
2.  Choose **Other type of secret**.
3.  Create the following key-value pairs. The key names are case-sensitive.
4.  Save the secret in the following format: `serverid/username`

Key - Password
Example Value - mySup3rS3cr3tPa55w0rd
Explanation: This password is used for SFTP and FTPS protocols

Key - Role
Example Value - arn:aws:iam::xxxxxxxxxxxx:role/AWSTransferAccessRole
Explanation: Use one of the Role ARNs you created for AWS Transfer users earlier. This will define what access the user has to S3

Optional:
Key - HomeDirectory
Example Value: /bucket/home/myhomedirectory
Explanation: The path to the user home directory. Not valid if HomeDirectoryDetails is used

Key - HomeDirectoryDetails
Example Value: [{"Entry": "/", "Target": "/ bucket/home/myhomedirectory"}]
Explanation: Logical folders mapping template. Not valid if HomeDirectory is used

Key - PublicKey
Example Value - ssh rsa public-key
Explanation: Comma separated list of public SSH keys (up to two keys)

Key - FTPPassword
Example Value - mySup3rS3cr3tFTPPa55w0rd
Explanation: This password is used for the FTP protocol

Key – AcceptedIpNetwork
Example Value – 192.168.1.0/24
Explanation: CIDR range of allowed source IP address for the client

![In the AWS Secrets Manager console, create a new secret by choosing Store a new secret.](https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2020/11/05/In-the-AWS-Secrets-Manager-console-create-a-new-secret-by-choosing-Store-a-new-secret..png)

If you would like to use different roles and passwords for each connection protocol, you can also prefix the **Role** and **Password** keys with the protocol. For example, for FTP specific parameters create `**FTPPassword**` and `**FTPRole**` keys.

**![If you would like to use different roles and passwords for each connection protocol, you can also prefix the Role and Password keys with the protocol.](https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2020/11/05/If-you-would-like-to-use-different-roles-and-passwords-for-each-connection-protocol-you-can-also-prefix-the-Role-and-Password-keys-with-the-protocol..png)**

## Now for the detail

The following steps explain what the AWS SAM package deploys.

### Amazon API Gateway endpoint

This is the glue between AWS Transfer Family and AWS Secrets Manager. The stack creates a _Get_ resource to match the required API path

/servers/serverId/users/username/config

_Method Request_ passes the following values through:

-   Query string: `serverId`, `username`, `protocol`, `sourceip`
-   Headers: `password`

_Integration Request_ maps the following variables into the Lambda function input parameters using the mapping template:

-   `username`
-   `password`
-   `serverId`
-   `protocol`
-   `sourceIp`

_Integration Response_ passes through the results returned by the Lambda function.

Finally, _Method Response_ returns a structured HTTP 200 response with the Lambda function output.

### AWS Lambda function

The Lambda function is a relatively simple Python 3.7 function. It takes the input parameters from the incoming Amazon API Gateway request and looks up the user name (in the format `serverId/username`) as a secret in AWS Secrets Manager.

The Lambda function implements this logic:

-   If the protocol is SFTP or FTPS and the password is provided by AWS Transfer Family, authenticate the user by comparing the password against the one stored within the secret.
-   If the protocol is FTP, authenticate the user by comparing the password against the alternative FTP password stored within the secret.
-   If the user tries to authenticate with an SSH key, no password is provided by AWS Transfer Family to the integration function. In this case, pass back any public SSH key stored in the secret to AWS Transfer Family so that it can perform the authorization checks.
-   If a client source IP range has been provided in the secret, check that the user is connecting from within that subnet range. The function supports a single IP CIDR range.

In all of the preceding cases, the function also returns the remaining parameters from the secret (Role, Home Directory, and so on). If a password check or source IP address check fails, then we return an empty response.

An AWS Lambda environment variable defines the AWS Region in which you should access Secret Manager. This defaults to the deployed AWS Region.

### IAM roles

The stack creates four IAM roles:

-   **TransferCWLoggingRole –** This role uses the AWS provided managed policy _AWSTransferLoggingAccess_ to enable AWS Transfer Family to create and send to a CloudWatch Logs stream.
-   **TransferApiCloudWatchLogsRole –** Allows the Amazon API Gateway endpoint to log to CloudWatch Logs.
-   **TransferApiInvokerAssumeRole –** Allows AWS Transfer Family to call the Amazon API Gateway endpoint as only services that you provide access to, can use this Amazon API Gateway endpoint.
-   **TransferLambdaExecutionRole –** Allows the Lambda function to execute, and provides the function with read-only access to Secrets Manager for secrets with the prefix _s-*_.

### AWS Transfer Family endpoint

AWS Transfer Family assumes an IAM role to access Amazon S3 on behalf of your connecting user. If you don’t already have a suitable IAM role for your users, start by creating one. This allows your users to access your S3 bucket for uploads or downloads. For more information, see the documentation on [creating an IAM role and policy](https://docs.aws.amazon.com/transfer/latest/userguide/requirements-roles.html).

When the AWS Transfer Family endpoint is created, the Amazon API Gateway endpoint is provided as a custom authentication provider. We also pass an IAM role that authorizes the service to invoke the Amazon API Gateway endpoint.

The AWS CLI command is shown as follows:

```bash
URL=https://APIGATEWAYID.execute-api.us-east-1.amazonaws.com/prod

INVOCATION_ROLE=arn:aws:iam::xxxxxxxxxxxx:role/xxxxx-TransferIdentityProviderRole-xxxxx 

LOGGING_ROLE= arn:aws:iam::xxxxxxxxxxxx:role/xxxxx-TransferCWLoggingRole-xxxxx 

aws --region us-east-1 transfer create-server --identity-provider-type API_GATEWAY --logging-role=${LOGGING_ROLE} --identity-provider-details "Url=${URL},InvocationRole=${INVOCATION_ROLE}" 
```

Bash

## Testing your setup

The AWS Transfer Family API provides a function to test whether the external authentication is working as expected. Swap in your server-id, plus the user name and password that you entered in AWS Secrets Manager:

```bash
aws transfer test-identity-provider --server-id "s-xxxxxxxxxx" --user-name charlie --user-password password
```

Bash

If you are using source and protocol checks, you can also include these on the call:

```bash
aws transfer test-identity-provider --server-id "s-xxxxxxxxxx" --user-name 
charlie --user-password password –source-ip "192.168.1.1" –server-protocol "FTP"
```

Bash

If all is working and you have passed through the correct user details, you should get the role information that you provided in Secrets Manager in the response field.

![If all is working and you have passed through the correct user details, you should get the role information that you provided in Secrets Manager in the response field.](https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2020/11/05/If-all-is-working-and-you-have-passed-through-the-correct-user-details-you-should-get-the-role-information-that-you-provided-in-Secrets-Manager-in-the-response-field.-1.png)

If the user is not authenticated (wrong user name or password), the test should return an error in the message field. If you don’t get the expected response at this stage, check the CloudWatch Log streams for AWS Transfer Family and the AWS Lambda function.

Finally, go ahead and try logging in with your external user details:

## Cleaning up

If you are done with the resources, do not forget to clean up and check for any permissions and IAM roles that are no longer required. Keep in mind that this this includes the AWS Transfer Family endpoint that you deployed. You can delete the stack from the AWS CloudFormation console.

## Summary

That’s it! You now have access to a fully managed, highly available transfer service supporting SFTP, FTPS, and FTP protocols.

The custom authentication pattern has enabled users to authenticate via passwords, but it also provides the flexibility to check other user attributes. For instance, you can check the source IP address to ensure certain users only connect from a specific location.

It is also common practice to use separate authentication and authorization for FTP users due to its unencrypted communication. This pattern provides the ability to check the client protocol and use separate authentication and authorization details for the older connection protocols.

Should you want to integrate with a different authentication provider, this guide and the associated AWS Lambda function code should provide you with a great starting point.








Link: https://aws.amazon.com/blogs/storage/enable-password-authentication-for-aws-transfer-family-using-aws-secrets-manager-updated/


video: https://www.youtube.com/watch?v=WH9FKyVjASo