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
## 3. Development

Download the complete code snippets from [Here](https://static.us-east-1.prod.workshops.aws/public/8a096b8e-0baf-42fb-aa6c-b3e736800167/assets/files.zip)

#### 1. Create S3 Bucket
An S3 bucket in the code snippets, be sure to provide your unique bucket name.
Python
```bash
import boto3

client = boto3.client('s3')

try:
    response = client.create_bucket(
        Bucket='your-bucket-name', # Add your bucket name here
        CreateBucketConfiguration={'LocationConstraint': 'us-west-2'}  # Specify your region if not using us-east-1
    )
    print(response)
except Exception as e:
    print(f"An error occurred: {e}")
```
When we run the code above and print the response from our API call, it should return something like this:
```bash
{
  "ResponseMetadata": {
    "RequestId": "QJ9031EF2V6RDGWK",
    "HostId": "CtoPHueAgF+IZsGRAdzhGIyikdJMV/91HgQurOfpfgaCGJmnfDMeuaxg0/q8iTRK+3cyuUiAUGM=",
    "HTTPStatusCode": 200,
    "HTTPHeaders": {
      "x-amz-id-2": "CtoPHueAgF+IZsGRAdzhGIyikdJMV/91HgQurOfpfgaCGJmnfDMeuaxg0/q8iTRK+3cyuUiAUGM=",
      "x-amz-request-id": "QJ9031EF2V6RDAEM",
      "date": "Tue, 12 Dec 2023 18:05:04 GMT",
      "location": "http://your-bucket-name.s3.amazonaws.com/",
      "server": "AmazonS3",
      "content-length": "0"
    },
    "RetryAttempts": 0
  },
  "Location": "http://your-bucket-name.s3.amazonaws.com/"
}
```
The "HTTPStatusCode": 200 indicates that our bucket was successfully created.

#### 2. Create IAM Role
The Lambda function requires an execution role with the permissions necessary for our workflow. To grant these permissions, we need to create a trust policy and permissions policy.

Python
```bash
import boto3
import json

iam = boto3.client('iam')

trust_policy = {
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

rekognition_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObjectTagging",
            ],
            "Resource": [
                "arn:aws:s3:::your-bucket-name/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "rekognition:DetectLabels",
            ],
            "Resource": "*"
        }
    ]
}

try:
    response = iam.create_role(
        RoleName='LambdaRekognitionRole',
        AssumeRolePolicyDocument=json.dumps(trust_policy)
    )
    role = response['Role']

    iam.put_role_policy(
        RoleName=role['RoleName'],
        PolicyName='RekognitionDetectLabelsPolicy',
        PolicyDocument=json.dumps(rekognition_policy)
    )

    print(role['Arn'])

except Exception as e:
    print(f"An error occurred: {e}")
```
Once you run this file, you should receive the ARN of an IAM role in this format after running the script above: arn:aws:iam::accountid:role/LambdaRekognitionRole. (Note that accountid should reflect your AWS account ID). Copy this role ARN to use in the next section.

#### 3. Create Lambda Function
The Lambda function includes a call to the Amazon Rekognition DetectLabels  API. This API returns an array of labels for the real-world objects detected in an image. We then use the PutObjectTagging  API to add those labels as tags to our S3 object.

By creating a Boto3 session that we can use to create clients for the S3 and Rekognition services. A session stores configuration state and allows you to create service clients and resources. 
Python
```bash
import boto3

session = boto3.session.Session()

rekognition = session.client('rekognition')
s3 = session.client('s3')

def handler(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    response = rekognition.detect_labels(
        Image={'S3Object': {'Bucket': bucket, 'Name': key}}
    )

    labels = []
    for label in response['Labels']:
        if label['Confidence'] > 80:
            labels.append(label['Name']) 

    labels = labels[:5]

    tag_set = []
    for i, label in enumerate(labels):
        tag_set.append({'Key': 'label'+str(i+1), 'Value': label})

    # Add tags to image 
    s3.put_object_tagging(
        Bucket=bucket,
        Key=key,
        Tagging={'TagSet': tag_set} 
    )
```
The Lambda function above processes an image from S3 and detects the top 5 labels with a confidence level above 80%. These labels are then added as tags to our S3 object.

#### Zip our Lambda file
In order to upload our code to lambda, we must first compress the file above into a .zip file which we will use in the next step. You can do this from the command line, for example: zip lambda_function.zip lambda_function.py.

#### Creating our Lambda function
We will use the CreateFunction  API to create our Lambda function, passing the code from lambda_function.zip using the ZipFile parameter. (Be sure to use your specific IAM role ARN. The accountid below should be your AWS account ID.)
Python
```bash
import boto3

client = boto3.client('lambda')

try:
    with open('lambda_function.zip', 'rb') as f: # Specify name of .zip file from last section
        file_contents = f.read()

    response = client.create_function(
        FunctionName='detect_image_uploaded_to_s3',
        Runtime='python3.11',
        Role='arn:aws:iam::accountid:role/LambdaRekognitionRole', # Input your role ARN here
        Handler='lambda_function.handler',
        Code={
            'ZipFile': file_contents
        }
    )

    print(response['FunctionArn'])

except Exception as e:
    print(f"An error occurred: {e}")
```
Once we run this file, it will output a FunctionARN which we will use in the next section.
#### 4. Create S3 Event Notification
Before creating the S3 event notification, we must allow our S3 bucket to invoke our function. To do that we will use the Lambda AddPermission  API:
Python
```bash
import boto3

client = boto3.client('lambda')

try:
    response = client.add_permission(
        FunctionName='detect_image_uploaded_to_s3',
        StatementId="s3invoke",
        Action="lambda:InvokeFunction",
        Principal="s3.amazonaws.com",
        SourceArn="arn:aws:s3:::your-bucket-name" # Add your bucket name here
    )
    print(response)
except Exception as e:
    print(f"An error occurred: {e}")
```
#### Add bucket notification configuration
We will use the PutBucketNotificationConfiguration  API to create an S3 event notification, which will trigger our Lambda function when a file is uploaded.

Note that we have specified the prefix filter images/ below, so that we only invoke our function when something is uploaded to S3 with that prefix. (You could also specify a suffix, for example if you only wanted this event to run when .jpg files are uploaded.)

For the LambdaFunctionArn value, be sure to specify the correct region and accountid values that correspond to your account.
Python
```bash
import boto3

s3 = boto3.client('s3')

try:
    response = s3.put_bucket_notification_configuration(
        Bucket='your-bucket-name', # Add your bucket name here
        NotificationConfiguration={
            "LambdaFunctionConfigurations": [
                {
                    # Input your actual Lambda ARN here:
                    "LambdaFunctionArn": "arn:aws:lambda:us-west-2:accountid:function:detect_image_uploaded_to_s3",
                    "Events": [
                        "s3:ObjectCreated:*"
                    ],
                    "Filter": {
                        "Key": {
                            "FilterRules": [
                                {
                                    "Name": "Prefix",
                                    "Value": "images/"
                                }
                            ]
                        }
                    }
                }
            ]
        }
    )
    print(response)
except Exception as e:
    print(f"An error occurred: {e}")
```
After running that code, we are ready to test our workflow in the next section!
