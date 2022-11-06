# Setting up online experiment using AWS S3
## The purpose of this essay
The main purpose of this guide is to help labs to run and manage online experiment in a convenient matter. Although there are infinitely many ways to run online applications, the way that’s presented in this manual is sufficient, relatively cheap and keep things organized and systematic.

## What you'll learn here
This guide is all about configuring your AWS account to host a static website using S3, save the collected data, and communicating with it through AWS JavaScript SDK (whether its your first time doing it or not).

## Before we begin
If its not the first time you configure your AWS account for this purpose, you can skip steps 1.2, 2, 3.

## Step 1: Create S3 buckets, and data folder
### 1.1	Create the experiment bucket
First, we'll create the experiment folder (in S3 the main folders called buckets) – the bucket in which our application/website/experiment code is located, and the one that hosts its run.
1.	On AWS console, search for S3, click it, and press on "create bucket".
2.	Name your bucket as you wish and choose the AWS region (for the most part, you can leave the default region). Note that S3 bucket name can't be change after the bucket has created, so choose a meaningful name.
3.	Uncheck the "Block all public access" box.
4.	Enable bucket versioning.
5.	Click "create bucket".
6.	Go to the properties tub inside your newly created bucket, scroll down, and enable static website hosting. Choose your index.html file (i.e., the main file of your experiment that makes it run. Traditionally called "index.html") and save the changes.
7.	Go to the permission tab, and in the bucket policy section, paste this code (Make sure to replace <span style="color:red;">BUCKET_NAME</span> with your actual bucket name!):
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicRead",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Resource": "arn:aws:s3:::BUCKET_NAME/*"
        }
    ]
}
```
That’s is! Your experiment bucket is good to go!

### 1.2 Create the "raw data" bucket
**Note:** If it's **not** your first experiment configuration on your AWS account, you shouldn't create this bucket, only the first one. Therefore, skip this part.
<br><br>Because of security considerations, it's a good practice to separate the data folder from the experiment folder. Therefore, the first AWS account configuration should include creating a data bucket.
1.	Follow steps 1-5 from section 1.1, while skipping step 6.
2.	Go to the permission tab, and in the CORS section, paste this code:
```
[
    {
        "AllowedHeaders": [ "*"],
        "AllowedMethods": ["PUT"],
        "AllowedOrigins": ["*"],
        "ExposeHeaders": []
    }
]
```
Now your raw data bucket is also ready, and that was (hopefully) the last time you've created this bucket.

### 1.3 Create data folder for the experiment
We want to create a folder (not a bucket, but an actual folder inside a bucket) in which all the subjects data will be saved. We already have our raw data bucket, so all we need to do is to create a new folder inside this bucket (it’s a good practice to name this folder with an indicative name that suggests its relation to the experiment bucket. One may even call them the exact same name).

## A couple of words before we proceed
The next two steps deal with complicated matters – authorization and authentication. This isn’t the place to spread the talk about those topics, but for some further reading, check those links: [AWS IAM Beginner Overview](https://www.youtube.com/watch?v=y8cbKJAo3B4), [Amazon Cognito Beginner Guide](https://www.youtube.com/watch?v=QEGo6ZoN-ao), [Secure API Access with Amazon Cognito](https://aws.amazon.com/blogs/compute/secure-api-access-with-amazon-cognito-federated-identities-amazon-cognito-user-pools-and-amazon-api-gateway/). Much more valuable information on those topics could be easily found online.
<br><br>**Note:** Those next two steps should be performed **only once** – at the initial configuration of your AWS account!

## Step 2: Create Cognito federated identity pool
1.	On AWS console, search for Cognito, click it, and press on "Manage Identity Pools".
2.	Click on "Create new identity pool", choose a name for it, and enable unauthenticated identities.
3.	In this stage, you may run into a problem – you wouldn’t be able to complete the identity pool creation because there are no roles for the authenticated / unauthenticated users. In this case, paste the following policy to both roles (if the roles don’t appear automatically at this point, you can do this manually through the IAM console):
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::BUCKET_NAME/*"
            ]
        }
    ]
}
```
**Note:** Make sure to replace the <span style="color:red;">BUCKET_NAME</span>.<br>
**Note:** We give this IAM users permission to only PUT stuff inside our bucket, but not getting things out of it (in particular, no one should be able to access the data).

## Step 3: Add Policy role in the IAM Console
If you didn’t came across any problem in the previous stage, than you can manually perform step 2.3 from the IAM console.

## Step 4 (optional): Create a CloudFront distribution
CloudFront enables our experiment website use a https protocol (instead of the default http protocol that S3 static website host uses), which is more secure one. In addition, due to its scalability, CloudFront enables the experiment to run more smoothly, even if the experiment contains some heavy preloading (more info here). To create a CloudFront distribution, follow these steps:
1.	On AWS console, search for CloudFront, click it, and press on "create new distribution".
2.	On the "origin domain" section, choose your experiment bucket.
3.	Under "Default cache behavior"->"Viewer"->"Viewer protocol policy", choose " Redirect HTTP to HTTPS".
4.	Click on "create distribution".
5.	Return to the CloudFront console and notice that your new distribution appears. In this line, under the "Last Modified" title, you'll see that the status is "Deploying". It might take some time until its ready to go (after deploying is over, the current date should appear there).
6.	After the distribution is all set up, the new link to the experiment is the address that appears under the title "Domain Name". **You should use only this link, and not the S3 default one!**
<br><br>**Note:** If updating the experiment bucket, it might take several hours to the CloudFront distribution to update itself with the new changes.

## Appendix: JavaScript function to save your data to S3 bucket
### Credit to Ari Dyckovsky.
```
const cognitoIdentityPool = "us-east-1:8g2ebsa3-59c6-7888-f548-s8n462b948d22";
const DATA_BUCKET = "raw-data";
const FATHER_DIRECTORY = "your-current-experiment-folder";


function saveDataToS3() {
    return

    id = subjID;
    csv = manipulateCsv();

    AWS.config.update({
        region: "us-east-1",
        credentials: new AWS.CognitoIdentityCredentials({
            IdentityPoolId: cognitoIdentityPool
        }),
    });

    const filename = `${FATHER_DIRECTORY}/${id}.csv`;

    const bucket = new AWS.S3({
        params: { Bucket: DATA_BUCKET },
        apiVersion: "2006-03-01",
    })

    const objParams = {
        Bucket: DATA_BUCKET,
        Key: filename,
        Body: csv
    }

    bucket.putObject(objParams, function(err, data) {
        if (err) {
            console.log("Error: ", err.message);
        } else {
            console.log("Data: ", data);
        }
    });
}

//Remove from data csv unnecessary columns and linse.
function manipulateCsv ()
{
    var csv = jsPsych.data.get();
    // Columns
    csv = csv.ignore('failed_video').ignore('failed_audio').ignore('failed_images').ignore('timeout').ignore('success').ignore('internal_node_id').ignore('question_order');
    // Lines
    csv = csv.filter([
                      {test_part: 'Train - Choice'},
                      {test_part: 'Block 1 - Choice'},
                      {test_part: 'Block 2 - Choice'},
                      {test_part: 'Block 3 - Choice'},
                      {test_part: 'Block 4 - Choice'},
                      {test_part: 'Corrects Estimation'},
                      ]);
    csv = csv.csv();
    return csv;
}
```
