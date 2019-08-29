# Show your emotions with Rekognition
<!-- TODO -->
<!-- Elaboration about the aws services used here S3 Rekognition Lambda -->
<!-- list the pre-requisites for the workshop -->
 - AWS services involved in this workshop
    - S3 - https://aws.amazon.com/s3/
    - Rekognition - https://aws.amazon.com/rekognition/
    - Lambda - https://aws.amazon.com/lambda/   (Workshop option)
 - Pre-requisites
    - AWS account (admin role recommended)
    - Cloud9 IDE

## Step 1 - Basic preparation
 - Install dependencies. Libraries we need are included in setup.sh. Run it to install.
 - Create an aws S3 bucket to store the image to be used in next steps. This bucket is also used to store processed image and enable a signed url to allow temporary access to the image from a user who doesn't have aws credentials. 
 - Create a rekognition collection to store the facial signature. A facial signature consists of feature vectors sampled by aws rekognition service from input image frame and this metadata can be used for matching faces and emotioanl analysis. AWS rekognition service groups the signatures about objects including faces into collections. 
```bash
# Install dependencies
# Make sure the console outputs "Dependencies installed successfully." 

./setup.sh

# Create your s3 bucket using command below, pick a globally unique bucket name. 
# This bucket name will be used in next steps. Name can be a mixture of lowercase letters and numbers.
# If successful, console will prompt: "make_bucket:<your bucket name>"
# Use command  'aws s3 ls' to verify the creation of bucket

aws s3 mb s3://<your_bucket_name>

# Create a rekognition pool 
# Console will prompt "StatusCode": 200 when successful

aws rekognition create-collection --collection-id <your_collection_name>

# Clone the github repo

git clone <repo>
```

## Step 2 - Upload an image to S3 bucket
 - Ready a photo and save it onto your local machine, make sure there is at lease one face in it. AWS rekognition service can index up to 100 faces at once, here we keep it simple by letting rekognition index the one prominent face. Detail shown in step 3.
 - Upload it to your Cloud9 IDE working directory: same directory where .py files resides
 - In case the Cloud9 'Upload Local Files" doesn't work, follow the steps below:
    - Step 1: Change "IMAGE_PATH" in 'convertImageBase64.py' and run it on local machine against the image file
    - Step 2: In Cloud9, open 'assembleImg.py', paste the base64 string into the designated place, and change the new image name, then run it. An image will be created.
 - Check if the image is in the working directory.
 - Give the command below to upload the image to the s3 bucket
```bash
# Parameter -i followed by path to the image and -b followed by the bucket name are required

python3 upload_to_s3.py -i <image_name> -b <bucket_name>
```

## Step 3 - Create a main function to index the image we put to S3 
 - Modify the variable names in index.py 
 - It performs tasks below sequentially: 
    - Indexing the face to AWS Rekognition service 
    - Process the metadata received from Rekognition service 
    - Print the resulsts of facial analysis to the console
    - Put a bounding box around the face on the image 
    - Send the image back to the s3 bucket 
 - Note: As we set "MaxFaces = 1", rekognition only return metadata about the most prominent face
 - Run it
```bash
python3 index.py
```

## Step 4 - Generate a signed url for users without aws credentials to temporarily access the image
 - A user who does not have AWS credentials or permission to access an S3 object can be granted temporary access by using a presigned URL
 - A presigned URL is generated by an AWS user who has access to the object. The generated URL is then given to the unauthorized user. The presigned URL can be entered in a browser or used by a program or HTML webpage. The credentials used by the presigned URL are those of the AWS user who generated the URL
 - A presigned URL remains valid for a limited period of time which is specified when the URL is generated
```bash
# Parameter -i followed by name of processed image and -b followed by the bucket name are required

python3 url_gen.py -i <image_name> -b <bucket_name>
```
 - Copy the url prompted on console and paste it to your browser

## (workshop option)
### Step 5 - Go serverless: create a lambda function
 - AWS Lambda lets you run code without provisioning or managing servers. You pay only for the compute time you consume - there is no charge when your code is not running.
    - Lambda features:
        - No server to manage
            - AWS Lambda automatically runs your code without requiring you to provision or manage servers. Just write the code and upload it to Lambda
        - Continous scaling
            - AWS Lambda automatically scales your application by running code in response to each trigger. Your code runs in parallel and processes each trigger individually, scaling precisely with the size of the workload
        - Subsecond metering 
            - With AWS Lambda, you are charged for every 100ms your code executes and the number of times your code is triggered. You don't pay anything when your code isn't running
 - Follow steps below to create a lambda function via management console:
    - Step 1 - Steup IAM role 
        - Go to the IAM dashboard by clicking Service the topleft corner and type in "IAM" and enter
        - Choose "Roles" --> "Create role" --> "AWS service" --> "Lambda" --> "Lambda" --> "Select your use case - lambda" --> "Next: Permission" --> "Create policy" --> Check "AmazonS3FullAccess" and "AmazonRekognitionFullAccess" --> "Next: Tags" --> "Next: Review" --> type a name into "Role name" --> "Create role"
    - Step 2 - Create a lambda function
        - Open aws management console, type "lambda" into the "Find Service" search bar and enter
        - Hit "Create function" then check "Author from scratch"
        - Enter a name for "Function name"
        - Choose Python 3.6 for Runtime
        - Choose "Use an existing role" then choose the IAM role created from last step
    - Step 3 - Add a event trigger to this lambda function following last step
        - Hit "+ Add trigger" button then select "S3"
        - Choose the bucket that has just been created and select "All object ctreate events" for Event type.
        - Check "Enable trigger" and hit "Add" button
        - Hit "Functions" at the topleft corner of the refreshed page
    - Step 4 - Edit the lambda function
        - Choose the lambda function from last step
        - Edit lambda_handler.py with some variable names changed (the file in the repo)
        - Scroll down to "Function code" area and replace the code in editor with the modified code in lambda_handler.py
        - Change "Timeout" to 10 sec in Basic setting section
        - Hit "Save" at topright corner
    - Step 5 - Setup environment for lambda
        - Note: the library cv2 used in this lambda function is not native to aws lambda runtime environment. So we need to set it up accordingly. 
        - In Cloud9 termianl, run: 
        ```bash
            docker run --rm -v $(pwd):/package tiivik/lambdazipper opencv-python 
            sudo zip -r9 opencv-python.zip lambda_handler.py
            aws lambda update-function-code --function-name <your_lambda_function_name> --zip-file fileb://opencv-python.zip
        ```
        - Change the content under "Handler" section to: "lambda_handler.lambda_handler" <-- without double quotes
        - Change "memory" to 512 mb in Basic setting section
        - Edit lambda_handler.py in Cloud9:
            - change variable names follow the comments
        - Then in Cloud9 termianl, run: 
        ```bash
            sudo zip -r9 opencv-python.zip lambda_handler.py
            aws lambda update-function-code --function-name <your_lambda_function_name> --zip-file fileb://opencv-python.zip
        ```
    - Step 6 - Trigger the lambda
        - Now we are ready to trigger the lambda function
        - Use command below:
        ```bash
        python3 upload_to_s3.py -i <image_name> -b <bucket_name>
        ```
    - Step 7 - Check the s3 bucket, open the lambda-processed image. There should be a bounding box around the face drew by the lambda function.