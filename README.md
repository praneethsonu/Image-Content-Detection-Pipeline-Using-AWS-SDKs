# Image Content Detection Pipeline Using AWS SDKs with Python
## Description
Highlights the cloud-based approach to recognizing objects in uploaded images using AWS tools.
These titles reflect the integration of key AWS services such as S3, Lambda, and Rekognition, which are aligned with the described workflow.
## We will implement the architecture below using AWS SDKs:
![image](https://github.com/user-attachments/assets/02af6f77-66f1-464d-a3e6-ae786d90c4c4)

The workflow involves the following steps:

1. Creating an S3 bucket for storing images and a Lambda function for processing those images.
2. Configuring S3 event notifications to invoke our Lambda function when a new image is uploaded to our bucket.
3. We use Amazon Rekognition in our Lambda function to detect what is in the uploaded image.
4. Adding tags to our S3 object based on the labels detected by Amazon Rekognition.

## Some topics before starting the project
1. AWS SDKs: A brief overview of the AWS SDKs, including installation, configuration, and authentication.
2. Development: Using SDKs to call service APIs and create resources.
3. Conclusion: Cleaning up the resources we created and summarizing takeaways.

## The Steps involved are:
### - Configuration
###- SDK Setuo
###- Development
###- Conclusion


