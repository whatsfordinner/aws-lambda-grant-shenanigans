# AWS Lambda Grant Shenanigans

When you provide environment variables to a Lambda function, they're encrypted using a KMS key. Either a customer-managed key that you provide or an AWS managed default key (with the alias `aws/lambda`). When environment variables are first defined, if the default key is used then Lambda creates a grant on that key letting the execution role use it for decrypting the environment variables.

But, if that role is deleted and then re-created, the grant is no longer valid! This is the same as other resource-based policies when the principal is removed but this is special because we never actually explicitly created that grant ourselves. This means the function will start failing for no apparent reason.

This repo helps set up a scenario that replicates this and shows the working parts.

## Setting Up

To start with, create a new stack with the provided CloudFormation template.

```
$ aws cloudformation create-stack \
  --stack-name grant-shenanigans \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  && aws cloudformation wait stack-create-complete \
  --stack-name grant-shenanigans
```

The template consists of 3 resources:  

* An IAM role `grant-shenanigans` with the Basic Execution Role managed policy  
* A Lambda function `grant-shenanigans` that uses the IAM role and has a single environment variable `foo`  
* A CloudWatch Log Stream that the function will write to

The log stream isn't important, it's just there to make sure the resource is deleted once we're done. Otherwise, the Lambda will create it automatically when it's first invoked and we won't be able to track it for deletion.  

Once the stack has been created you'll be able to invoke the function to make sure it's working properly

```
$ aws lambda invoke \
  --function-name grant-shenanigans \
  outfile
```

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}

```

Additionally, let's check out the grants on the `aws/lambda` key. First we need to find what key that alias points to:

```
$ aws kms list-aliases \
  --query "Aliases[?AliasName=='alias/aws/lambda'].TargetKeyId"
```

```json
{
    "5c466b7d-7ad6-4c88-9fe6-bb787707c55a"
}
```

And now check the grants. This can only be done via the CLI or API:

```
$ aws kms list-grants \
  --key-id 5c466b7d-7ad6-4c88-9fe6-bb787707c55a \
  --query "Grants[?Name=='grant-shenanigans']"
```

```json
[
    {
        "KeyId": "arn:aws:kms:ap-southeast-2:123456789012:key/5c466b7d-7ad6-4c88-9fe6-bb787707c55a",
        "GrantId": "4056e6728604a352bb666729c88fcef50b183c8d87d019ea4c8d1441f05c1ebd",
        "Name": "grant-shenanigans",
        "CreationDate": "2021-09-23T21:31:55+10:00",
        "GranteePrincipal": "arn:aws:sts::123456789012:assumed-role/grant-shenanigans/grant-shenanigans",
        "RetiringPrincipal": "arn:aws:sts::123456789012:assumed-role/grant-shenanigans/grant-shenanigans",
        "IssuingAccount": "arn:aws:iam::123456789012:root",
        "Operations": [
            "Decrypt",
            "RetireGrant"
        ],
        "Constraints": {
            "EncryptionContextEquals": {
                "aws:lambda:FunctionArn": "arn:aws:lambda:ap-southeast-2:123456789012:function:grant-shenanigans"
            }
        }
    }
]
```

We never explicitly asked for this grant, it was done in the background by Lambda. We can see this in CloudTrail, as well:

```
$ aws cloudtrail lookup-events \
--lookup-attributes AttributeKey=EventName,AttributeValue=CreateGrant \
--query 'Events[].CloudTrailEvent' \
--output text \
  | jq 'select(.requestParameters.keyId | contains("5c466b7d-7ad6-4c88-9fe6-bb787707c55a"))'
```

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAIJ5Y6X73VIQFYZVMS:botocore-session-1632396667",
    "arn": "arn:aws:sts::123456789012:assumed-role/admin-role/botocore-session-1632396667",
    "accountId": "123456789012",
    "accessKeyId": "ASIAIROISRJEHT4UKGBQ",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROAIJ5Y6X73VIQFYZVMS",
        "arn": "arn:aws:iam::123456789012:role/admin-role",
        "accountId": "123456789012",
        "userName": "admin-role"
      },
      "webIdFederationData": {},
      "attributes": {
        "creationDate": "2021-09-23T11:31:20Z",
        "mfaAuthenticated": "false"
      }
    },
    "invokedBy": "lambda.amazonaws.com"
  },
  "eventTime": "2021-09-23T11:31:55Z",
  "eventSource": "kms.amazonaws.com",
  "eventName": "CreateGrant",
  "awsRegion": "ap-southeast-2",
  "sourceIPAddress": "lambda.amazonaws.com",
  "userAgent": "lambda.amazonaws.com",
  "requestParameters": {
    "keyId": "arn:aws:kms:ap-southeast-2:123456789012:key/5c466b7d-7ad6-4c88-9fe6-bb787707c55a",
    "operations": [
      "Decrypt",
      "RetireGrant"
    ],
    "constraints": {
      "encryptionContextEquals": {
        "aws:lambda:FunctionArn": "arn:aws:lambda:ap-southeast-2:123456789012:function:grant-shenanigans"
      }
    },
    "retiringPrincipal": "arn:aws:sts::123456789012:assumed-role/grant-shenanigans/grant-shenanigans",
    "granteePrincipal": "arn:aws:sts::123456789012:assumed-role/grant-shenanigans/grant-shenanigans"
  },
  "responseElements": {
    "grantId": "4056e6728604a352bb666729c88fcef50b183c8d87d019ea4c8d1441f05c1ebd"
  },
  "requestID": "1c310e5c-878d-4910-8f6f-87af9e1d229e",
  "eventID": "fd50e729-59fb-4f5a-bb46-6267852a7547",
  "readOnly": false,
  "resources": [
    {
      "accountId": "123456789012",
      "type": "AWS::KMS::Key",
      "ARN": "arn:aws:kms:ap-southeast-2:123456789012:key/5c466b7d-7ad6-4c88-9fe6-bb787707c55a"
    }
  ],
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "123456789012",
  "eventCategory": "Management"
}

```

So now we know that we know everything's set up properly, let's break it.

## Deleting the Role

There's a lot of reasons that an IAM role might get deleted unintentionally. By accident in the console, over-zealous pruning of IAM, even a result of infrastructure-as-code activities.

```
$ aws iam detach-role-policy \
  --role-name grant-shenanigans \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole \
  && aws iam delete-role --role-name grant-shenanigans
```

## Recreating the Role

Similarly, the role can be recreated in a lot of ways. Because we're using CloudFormation, the [recommended action](https://aws.amazon.com/premiumsupport/knowledge-center/failing-stack-updates-deleted/) when a resource is deleted out of band is to recreate it as exactly as possible. We can see how this works in action with drift detection.

Drift detection is way easier to parse in the console so let's make our way over there and Detect Stack Drift:

![A screenshot of the AWS CloudFormation console showing the drift status of our example stack. The Role is marked as deleted.](img/drift1.png)

Alright, let's recreate the role as exactly as possible:

```
$ aws iam create-role \
  --role-name grant-shenanigans \
  --assume-role-policy-document file://assume_role_policy.json
```

```json
{
    "Role": {
        "Path": "/",
        "RoleName": "grant-shenanigans",
        "RoleId": "AROAWJFTKRGSJLRRX5PRS",
        "Arn": "arn:aws:iam::123456789012:role/grant-shenanigans",
        "CreateDate": "2021-09-23T12:13:17+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "lambda.amazonaws.com"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}
```

At this point, drift detection should show that the role is modified because the managed policy is missing from it:

![A screenshot of the AWS CloudFormation console showing the drift status of our example stack. The Role is marked as modified.](img/drift2.png)

![A screenshot of the AWS CloudFormation console showing the drift status of the example role. The managed policy ARN is missing from the role.](img/drift3.png)

```
$ aws iam attach-role-policy \
  --role-name grant-shenanigans \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

Now, drift detection shows that everything is as expected. By all accounts, we've recreated the role as expected:

![A screenshow of the AWS CloudFormation console showing the drift status of the example role. All resources are in sync.](img/drift4.png)

Let's try invoking the lambda again!

```
$ aws lambda invoke \
  --function-name grant-shenanigans \
  outfile
```

We'll get an error. But, it's actually very, very rare that you would invoke a lambda this way. So, before we look at the error, let's have a look at some of the other ways we can try to identify that the problem is occurring.

## Identifying the Issue

Firstly, let's check the CloudWatch metrics for our lambda:

![A screenshot of the AWS CloudWatch metrics console showing the invocation stats for the function. Most recently there was a sum of 3 errors and no metric for invocations.](img/cloudwatch1.png)

We can see the error show up but there's no matching invocation. This means that our code has never actually run. We can confirm this by checking for the log events as well. In the log group, there will be no log from the invocation we just tried to make:

![A screenshot of the AWS CloudWatch logs console showing the invocation logs for the function. There's only a single log stream with the last event occurring the last time the function was successful.](img/cloudwatch2.png)

![A screenshot of the AWS CloudWatch logs console showing the contents of the only log stream for the function. The only logs are from the last successful invocation.](img/cloudwatch3.png)

At this point, we know that the last changes made were to the execution role. Our best bet is to check CloudTrail for any activity by that role:

First, let's check to see if the role was successfully assumed:

```
$ aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=grant-shenanigans \
  --query 'Events[].CloudTrailEvent' \
  --output text \
  | jq 'select(.eventName == "AssumeRole") | select(.userAgent == "lambda.amazonaws.com")'
```

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "AWSService",
    "invokedBy": "lambda.amazonaws.com"
  },
  "eventTime": "2021-09-23T12:26:08Z",
  "eventSource": "sts.amazonaws.com",
  "eventName": "AssumeRole",
  "awsRegion": "ap-southeast-2",
  "sourceIPAddress": "lambda.amazonaws.com",
  "userAgent": "lambda.amazonaws.com",
  "requestParameters": {
    "roleArn": "arn:aws:iam::123456789012:role/grant-shenanigans",
    "roleSessionName": "grant-shenanigans"
  },
  "responseElements": {
    "credentials": {
      "accessKeyId": "ASIAWJFTKRGSCOYD3HRW",
      "sessionToken": "",
      "expiration": "Sep 24, 2021, 12:26:08 AM"
    }
  },
  "requestID": "6bdbd87c-641f-4e58-8ef4-518e79cd07d9",
  "eventID": "4de7689a-a98f-45d0-a488-332a50c50a04",
  "readOnly": true,
  "resources": [
    {
      "accountId": "123456789012",
      "type": "AWS::IAM::Role",
      "ARN": "arn:aws:iam::123456789012:role/grant-shenanigans"
    }
  ],
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "eventCategory": "Management",
  "recipientAccountId": "123456789012",
  "sharedEventID": "08690261-5f15-4df6-a2dd-61c354e48141"
}
```

We can see that Lambda was able to successfully assume the role _and_ the event includes an AccessKey which we can use to track all the API calls that were made while the function was attempting to run.

```
$ aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=ASIAWJFTKRGSCOYD3HRW \
  --query 'Events[].CloudTrailEvent' \
  --output text | jq
```

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAWJFTKRGSJLRRX5PRS:grant-shenanigans",
    "arn": "arn:aws:sts::123456789012:assumed-role/grant-shenanigans/grant-shenanigans",
    "accountId": "123456789012",
    "accessKeyId": "ASIAWJFTKRGSCOYD3HRW",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "principalId": "AROAWJFTKRGSJLRRX5PRS",
        "arn": "arn:aws:iam::123456789012:role/grant-shenanigans",
        "accountId": "123456789012",
        "userName": "grant-shenanigans"
      },
      "webIdFederationData": {},
      "attributes": {
        "creationDate": "2021-09-23T12:26:08Z",
        "mfaAuthenticated": "false"
      }
    }
  },
  "eventTime": "2021-09-23T12:26:08Z",
  "eventSource": "kms.amazonaws.com",
  "eventName": "Decrypt",
  "awsRegion": "ap-southeast-2",
  "sourceIPAddress": "1.1.1.1",
  "userAgent": "aws-internal/3 aws-sdk-java/1.12.64 Linux/4.14.243-185.433.amzn2.x86_64 OpenJDK_64-Bit_Server_VM/25.302-b08 java/1.8.0_302 vendor/Oracle_Corporation cfg/retry-mode/standard",
  "errorCode": "AccessDenied",
  "errorMessage": "User: arn:aws:sts::123456789012:assumed-role/grant-shenanigans/grant-shenanigans is not authorized to perform: kms:Decrypt on resource: arn:aws:kms:ap-southeast-2:123456789012:key/5c466b7d-7ad6-4c88-9fe6-bb787707c55a",
  "requestParameters": null,
  "responseElements": null,
  "requestID": "a009763a-8429-4cc5-87da-8250d4789772",
  "eventID": "ea82ecfd-24a7-4fe7-94de-493770c51e23",
  "readOnly": true,
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "123456789012",
  "eventCategory": "Management"
}
```

Now we can see an error:

> User: arn:aws:sts::123456789012:assumed-role/grant-shenanigans/grant-shenanigans is not authorized to perform: kms:Decrypt on resource: arn:aws:kms:ap-southeast-2:123456789012:key/5c466b7d-7ad6-4c88-9fe6-bb787707c55a

That key ID is familiar, but let's double check

```
$ aws kms describe-key \
  --key-id 5c466b7d-7ad6-4c88-9fe6-bb787707c55a
```

```json
{
    "KeyMetadata": {
        "AWSAccountId": "123456789012",
        "KeyId": "5c466b7d-7ad6-4c88-9fe6-bb787707c55a",
        "Arn": "arn:aws:kms:ap-southeast-2:123456789012:key/5c466b7d-7ad6-4c88-9fe6-bb787707c55a",
        "CreationDate": "2020-07-24T22:08:35.285000+10:00",
        "Enabled": true,
        "Description": "Default master key that protects my Lambda functions when no other key is defined",
        "KeyUsage": "ENCRYPT_DECRYPT",
        "KeyState": "Enabled",
        "Origin": "AWS_KMS",
        "KeyManager": "AWS",
        "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT",
        "KeySpec": "SYMMETRIC_DEFAULT",
        "EncryptionAlgorithms": [
            "SYMMETRIC_DEFAULT"
        ],
        "MultiRegion": false
    }
}
```

As expected, it's the Amazon managed key for Lambda! Let's have a look at that grant:

```
$ aws kms list-grants \
  --key-id 5c466b7d-7ad6-4c88-9fe6-bb787707c55a \
  --query "Grants[?Name=='grant-shenanigans']"
```

```json
[
    {
        "KeyId": "arn:aws:kms:ap-southeast-2:123456789012:key/5c466b7d-7ad6-4c88-9fe6-bb787707c55a",
        "GrantId": "4056e6728604a352bb666729c88fcef50b183c8d87d019ea4c8d1441f05c1ebd",
        "Name": "grant-shenanigans",
        "CreationDate": "2021-09-23T21:31:55+10:00",
        "GranteePrincipal": "AROAWJFTKRGSL363D6SWQ:grant-shenanigans",
        "RetiringPrincipal": "AROAWJFTKRGSL363D6SWQ:grant-shenanigans",
        "IssuingAccount": "arn:aws:iam::123456789012:root",
        "Operations": [
            "Decrypt",
            "RetireGrant"
        ],
        "Constraints": {
            "EncryptionContextEquals": {
                "aws:lambda:FunctionArn": "arn:aws:lambda:ap-southeast-2:123456789012:function:grant-shenanigans"
            }
        }
    }
]
```

The `GranteePrincipal` and `RetiringPrincipal` no longer use the ARN of the role, instead they use the unique ID for the role that was generated when it was first created. If you look at the output JSON from when we recreated the role you can see the `RoleID` attribute and see that it's different. That's why the grant no longer applies.

Let's recreate that grant:

```
$ aws kms create-grant \
  --key-id 5c466b7d-7ad6-4c88-9fe6-bb787707c55a \
  --grantee-principal arn:aws:iam::123456789012:role/grant-shenanigans \
  --retiring-principal arn:aws:iam::123456789012:role/grant-shenanigans \
  --operations Decrypt RetireGrant 

An error occurred (AccessDeniedException) when calling the CreateGrant operation: User: arn:aws:sts::123456789012:assumed-role/admin-role/botocore-session-1632399481 is not authorized to perform: kms:CreateGrant on resource: arn:aws:kms:ap-southeast-2:123456789012:key/5c466b7d-7ad6-4c88-9fe6-bb787707c55a

```

Oh. That's because we aren't allowed to manage grants on [AWS Managed Keys](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#aws-managed-cmk):

> You can view the AWS managed keys in your account, view their key policies, and audit their use in AWS CloudTrail logs. However, you cannot manage these KMS keys, rotate them, or change their key policies.

## Fixing It

At this point, you can fix the function either by changing the execution role back and forth or recreating the function. I would opt for recreating the function because the lambda is as broken as it gets, anyway.

```
$ aws cloudformation delete-stack \
  --stack-name grant-shenanigans \
  && aws cloudformation wait stack-delete-complete \
  --stack-name grant-shenanigans
$ aws cloudformation create-stack \
  --stack-name grant-shenanigans \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  && aws cloudformation wait stack-create-complete \
  --stack-name grant-shenanigans
```

Depending on your setup, this is going to be more or less involved. E.g. if you have datastores or other stateful resources in your stack then you're going to want to do a targeted delete and recreate of just the function rather than just blowing away the whole stack. Deleting and recreating the stack is just for the purposes of this demo.

Then try to invoke the lambda:

```
$ aws lambda invoke \
  --function-name grant-shenanigans \
  outfile
```

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

## Teardown

Delete the CloudFormation stack when you're done!

```
$ aws cloudformation delete-stack \
  --stack-name grant-shenanigans
```