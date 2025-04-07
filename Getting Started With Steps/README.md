## 1. Configuration

### Defining profiles

I recommend [installing the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and running the [aws configure](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/configure/index.html) command to easily configure your profile. For example:

```bash
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-west-2
Default output format [None]: json
```
Here are some examples of what a typical AWS credentials file and config file might look like:
credentials file (~/.aws/credentials):
```bash
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```
## 2. SDK Setup

The AWS SDK for Python is also known as Boto3. The Botocore  library provides the low-level functionality shared between the Python SDK and AWS CLI.

You can install the latest Boto3 version via pip:
```bash
pip install boto3
```
If you now run pip show boto3 then it should output something like this:
```bash
Name: boto3
Version: 1.34.1
Summary: The AWS SDK for Python
Home-page: https://github.com/boto/boto3
Author: Amazon Web Services
...
```
Confirming authorization
The GetCallerIdentity API  returns details about the IAM user or role whose credentials are used to call the operation. This way we can confirm that we are using the correct account.
Python
```bash
import boto3

try:
    client = boto3.client('sts')
    response = client.get_caller_identity()
    print(response)
except Exception as e:
    print(f"An error occurred: {e}")
```
Running that script should output our expected identity info:
```bash
{
    'Account': '123456789012',
    'Arn': 'arn:aws:iam::123456789012:user/Alice',
    'UserId': 'AKIAI44QH8DHBEXAMPLE',
    'ResponseMetadata': {
        '...': '...',
    },
}
```

