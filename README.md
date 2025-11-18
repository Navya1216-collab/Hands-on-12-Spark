# Hands-on-12-Spark-on-AWS
# Serverless Spark ETL Pipeline on AWS

This project is a hands-on assignment demonstrating a fully automated, event-driven serverless data pipeline on AWS.

The pipeline automatically ingests raw CSV product review data, processes it using a Spark ETL job, runs analytical SQL queries on the data, and saves the aggregated results back to S3.

---

## 📊 Project Overview

The core problem this project solves is the need for manual data processing. In a typical scenario, data lands in S3 and waits for a data engineer to run a job. This project automates that entire workflow.

**The process is as follows:**
1.  A raw `reviews.csv` file is uploaded to an S3 "landing" bucket.
2.  The S3 upload event instantly triggers an **AWS Lambda** function.
3.  The Lambda function starts an **AWS Glue ETL job**.
4.  The Glue job (running a PySpark script) reads the CSV, cleans it, and runs multiple Spark SQL queries to generate analytics (e.g., average ratings, top customers).
5.  The final, aggregated results are written as Parquet files to a separate S3 "processed" bucket.

---

## 🏗️ Architecture



**Data Flow:**
`S3 (Upload) -> Lambda (Trigger) -> AWS Glue (Spark Job) -> S3 (Processed Results)`

---

## 🛠️ Technology Stack

* **Data Lake:** Amazon S3
* **ETL (Spark):** AWS Glue
* **Serverless Compute:** AWS Lambda
* **Data Scripting:** PySpark (Python + Spark SQL)
* **Security:** AWS IAM (Identity and Access Management)

---

## 🔧 Setup and Deployment

Follow these steps to deploy the pipeline in your own AWS account.

### 1. Prerequisites
* An AWS Account (Free Tier is sufficient)
* Basic knowledge of S3, IAM, Lambda, and Glue

### 2. Create S3 Buckets
Create two S3 buckets with globally unique names:
* `handsonfinallanding`: This is where you will upload your raw data.
* `handsonfinalprocessed`: This is where the processed data and query results will be stored.
* Landing Bucket (Raw Data Bucket)
Bucket Name: handsonfinallandingnavv
Region: us-east-2 (Ohio)

This bucket is used as the entry point of the entire pipeline.
Whenever a new reviews.csv file is uploaded to this bucket, it immediately triggers the Lambda function through an S3 ObjectCreated event.

Purpose of This Bucket:

Stores raw, unprocessed data uploaded by the user or automated system

Acts as the event source for AWS Lambda

Initiates the ETL workflow automatically

Maintains a clean “landing zone” similar to real-world data lake architectures

Why it is needed:

This bucket allows us to simulate a real production scenario where external applications or services drop new files into a landing zone, and downstream components process the data automatically without human intervention.

2️⃣ Processed Bucket (Analytics Output Bucket)
Bucket Name: handsonfinalprocessednav
Region: us-east-1 (N. Virginia)

* <img width="1899" height="935" alt="image" src="https://github.com/user-attachments/assets/20d30d4b-eee8-41ec-8db9-45bed80d3d58" />
<img width="1919" height="946" alt="image" src="https://github.com/user-attachments/assets/8ae6a462-2ec0-4a68-b530-da7552555d2f" />



### 3. Create IAM Role for AWS Glue
Your Glue job needs permission to read from and write to S3.

1.  Go to the **IAM** service.
2.  Create a new **Role**.
3.  Select **AWS service** as the trusted entity and choose **Glue** as the use case.
4.  Attach the `AWSGlueServiceRole` managed policy.
5.  Attach the `AmazonS3FullAccess` policy (for this demo) or a more restrictive policy that only grants access to your two buckets.
6.  Name the role `AWSGlueServiceRole-Reviews` and create it.

Create IAM Role for AWS Glue — Explanation for README

AWS Glue requires an IAM role so it can securely access the resources it needs during the ETL process. Since Glue does not run as a user, it must assume a role that grants permissions to read input data, write processed output, and interact with AWS services.

In this project, we created a dedicated IAM role named:

AWSGlueServiceRole-Reviews

Why This IAM Role Is Needed

The Glue ETL job performs multiple operations that require permissions:

Read raw data from:
s3://handsonfinallandingnavv

Write processed output to:
s3://handsonfinalprocessednav

Log job execution details to Amazon CloudWatch

Glue logs Spark driver/executor output, errors, and job status.
This also requires IAM permissions.

Execute Spark and ETL operations inside AWS Glue

Glue needs its default service permissions for job execution, script loading, and cluster provisioning.

Without this IAM role, the Glue job cannot run, read data, or write output.

Policies Attached to the Role

To allow the Glue job to run successfully, we attached:

AWSGlueServiceRole (Managed Policy)

This built-in AWS policy provides:

Glue job execution permissions

Logging permissions

Spark infrastructure access

Internal Glue metadata access

AmazonS3FullAccess (Demo-Friendly)

This allows the Glue job to:

Read raw CSV files from the landing bucket

Write processed Parquet results into the output bucket

Create directories (“folders”) inside S3

In a real production environment, this policy would be restricted to only your two buckets, but for this assignment the full-access policy simplifies setup and avoids permission errors.
   <img width="1916" height="920" alt="image" src="https://github.com/user-attachments/assets/57766085-4871-472b-9739-49dbc08dcd9a" />


### 4. Create the AWS Glue ETL Job
1.  Go to the **AWS Glue** service.
2.  In the navigation pane, click on **ETL jobs**.
3.  Select the **Spark script editor** option to create a new job.
4.  Paste the contents of `src/glue_job_script.py` into the editor.
5.  Go to the **Job details** tab.
6.  Set the **Name** to `process_reviews_job`.
7.  Select the `AWSGlueServiceRole-Reviews` **IAM Role** you created in the previous step.
8.  Save the job.

The AWS Glue ETL job is responsible for running the PySpark script that processes raw review data. This job reads the CSV file from the landing bucket, performs data cleaning, executes multiple Spark SQL analytics queries, and writes the final results to the processed bucket.

By creating a Glue job named process_reviews_job and assigning it the AWSGlueServiceRole-Reviews IAM role, we ensure that the job has the required permissions to access S3 and execute the ETL workflow. The Spark script editor is used to paste and run the full ETL script (src/glue_job_script.py) that powers the entire serverless analytics pipeline.
<img width="1919" height="928" alt="image" src="https://github.com/user-attachments/assets/c8088f49-672c-4278-98ba-22ad3f3887ca" />


> **Note:** The script is already configured to use the `handsonfinallanding` and `handsonfinalprocessed` buckets.

### 5. Create the Lambda Trigger Function
This function will start the Glue job when a file is uploaded.

1.  Go to the **AWS Lambda** service and **Create function**.
2.  Select **Author from scratch**.
3.  Set the **Function name** to `start_glue_job_trigger`.
4.  Set the **Runtime** to **Python 3.10** (or any modern Python runtime).
5.  **Permissions:** Under "Change default execution role," select **Create a new role with basic Lambda permissions**. This role will be automatically named.
6.  Create the function.

The Lambda function acts as the event-driven orchestrator of the pipeline. Whenever a new reviews.csv file is uploaded to the landing bucket, this function is automatically triggered by S3. Its only job is to start the AWS Glue ETL job (process_reviews_job) without any manual intervention. Creating the function with basic Lambda permissions allows it to run securely while keeping the architecture fully serverless and automated.
   <img width="1919" height="917" alt="image" src="https://github.com/user-attachments/assets/890593c2-0503-46ce-acc5-1eb4db742dee" />


#### 5a. Add Lambda Code
Paste the contents of `src/lambda_function.py` into the code editor. Make sure the `GLUE_JOB_NAME` variable matches the name of your Glue job (`process_reviews_job`).
The Lambda function needs custom Python code to start the Glue ETL job whenever it is triggered by an S3 upload event. By pasting the contents of src/lambda_function.py into the Lambda code editor and ensuring that the GLUE_JOB_NAME variable matches process_reviews_job, we link the function directly to the Glue job. This makes Lambda responsible for launching the ETL process automatically whenever new data arrives.
<img width="1919" height="987" alt="image" src="https://github.com/user-attachments/assets/7b265bb6-f01b-4968-a21e-c00ad7811af1" />


#### 5b. Add Lambda Permissions
The new Lambda role needs permission to start a Glue job.
1.  Go to the function's **Configuration** > **Permissions** tab and click the role name.
2.  In the IAM console, click **Add permissions** > **Create inline policy**.
3.  Use the JSON editor and paste the following policy:
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "glue:StartJobRun",
                "Resource": "*"
            }
        ]
    }
    ```
4.  Name the policy `Allow-Glue-StartJobRun` and save it.
By default, the Lambda function does not have permission to start a Glue job. To enable this, we attach an inline IAM policy to the Lambda execution role allowing the glue:StartJobRun action. This permission is essential because it gives Lambda the authority to trigger the ETL process in Glue whenever new data arrives. Naming the policy Allow-Glue-StartJobRun keeps the setup organized and easy to identify later
<img width="1916" height="983" alt="image" src="https://github.com/user-attachments/assets/607c0836-5e82-46a8-b31a-6ee92839bf1e" />


#### 5c. Add the S3 Trigger
1.  Go back to your Lambda function's main page.
2.  Click **Add trigger**.
3.  Select **S3** as the source.
4.  Select your `handsonfinallanding` bucket.
5.  Set the **Event type** to `s3:ObjectCreated:*` (or "All object create events").
6.  Acknowledge the recursive invocation warning and click **Add**.
The S3 trigger connects your landing bucket to the Lambda function, enabling true automation. When a new reviews.csv file is uploaded to the handsonfinallandingnavv bucket, S3 immediately invokes the Lambda function using an ObjectCreated event. This ensures the ETL pipeline runs automatically without any manual execution, making the entire workflow event-driven and fully serverless.
<img width="1915" height="935" alt="image" src="https://github.com/user-attachments/assets/954591f7-007f-4327-9d27-1c83a09332da" />

---

## 🚀 How to Run the Pipeline

Your pipeline is now fully deployed and automated!

1.  Take the sample `reviews.csv` file from the `data/` directory.
2.  Upload `reviews.csv` to the root of your `handsonfinallanding` S3 bucket.
3.  This will trigger the Lambda, which in turn starts the Glue job.
4.  You can monitor the job's progress in the **AWS Glue** console under the **Monitoring** tab.
Running the pipeline is as simple as uploading the reviews.csv file to the landing bucket. As soon as the file is uploaded, S3 triggers Lambda, which starts the Glue ETL job automatically. Glue then processes the data, runs all SQL analytics, and saves the final results into the processed bucket. This fully automated workflow ensures new data is transformed and ready for analysis without any manual intervention.
---
<img width="1917" height="937" alt="image" src="https://github.com/user-attachments/assets/b9637152-4dcd-417b-be65-92d2cc3a5440" />
<img width="1911" height="947" alt="image" src="https://github.com/user-attachments/assets/1e7627b7-4541-4ce8-9521-d225752a2cdf" />




## 📈 Query Results

After the job (which may take 2-3 minutes to run), navigate to your `handsonfinalprocessed` bucket. You will find the results in the `Athena Results/` folder, organized into sub-folders for each query:

* `s3://handsonfinalprocessed/Athena Results/daily_review_counts/`
* `s3://handsonfinalprocessed/Athena Results/top_5_customers/`
* `s3://handsonfinalprocessed/Athena Results/rating_distribution/`

You will also find the complete, cleaned dataset in `s3://handsonfinalprocessed/processed-data/`.
After the Glue ETL job completes, all analytical results are written as Parquet files into the processed bucket. Each query output is organized into its own folder—for example, daily review counts, top customers, and rating distributions. This clear folder structure makes it easy to explore, validate, or query the processed data further using tools like AWS Athena. The final cleaned dataset is also stored for downstream analytics.
<img width="1911" height="947" alt="image" src="https://github.com/user-attachments/assets/0439ca8e-5b70-483f-8945-555ad428c9c3" />
<img width="1906" height="977" alt="image" src="https://github.com/user-attachments/assets/c1b56da7-a434-49ce-8b44-6b94915e2a25" />
<img width="1915" height="943" alt="image" src="https://github.com/user-attachments/assets/1f630734-0cab-44ec-82ab-38c854d6bf7a" />
<img width="1908" height="942" alt="image" src="https://github.com/user-attachments/assets/274f9711-7e47-4d0c-b790-032ff683ffed" />
<img width="1917" height="953" alt="image" src="https://github.com/user-attachments/assets/1f7b9218-1e62-416d-84e4-0423f588f5b1" />





---
## 🧹 Cleanup

To avoid any future charges (especially if you're on the Free Tier), be sure to delete the resources you created:
1.  Empty and delete the `handsonfinallanding` and `handsonfinalprocessed` S3 buckets.
2.  Delete the `start_glue_job_trigger` Lambda function.
3.  Delete the `process_reviews_job` Glue job.
4.  Delete the `AWSGlueServiceRole-Reviews` IAM role.
