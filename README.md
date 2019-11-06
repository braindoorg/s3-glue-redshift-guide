# Daily GA Transaction Data To Redshift: AWS Setup Guide

The following instructions assume you are logged into AWS with an account full permission to list, modify, and create services.

## Primary Services Involved:
- IAM
- S3
- Glue
- Redshift
- Lambda



***

## IAM

Create an IAM Role for the glue-workflow-initiating lambda function so it has permissions to:
- detect changes to S3
- execute a python script that uses the boto3 API to start a Glue workflow
- write logs to CloudWatch and CloudTrail

### Setup Instructions:
  1. Navigate to the IAM dashboard: https://console.aws.amazon.com/iam/home
  2. Click **Roles** in the sidebar
  3. Click **Create role** button
  4. Under **Choose the service that will use this role**, click **Lambda** > *Next*
  5. Search for, and check two policies: `AWSLambdaBasicExecutionRole` and `AWSGlueServiceRole` > *Next*
  6. (optional) add a Key/Value pair per your current AWS tag taxonomy > *Next*
  7. Add Role name `lambda_lambda-s3-glue` > *Create role*



***



## S3

S3 will be used as the primary data store for Glue ETL jobs. It will also be used to store temporary, auto-generated files from different Amazon services as they run.

### Setup Instructions:
  1. Navigate to the S3 dashboard: https://s3.console.aws.amazon.com/s3/home
  2. Click **+ Create bucket**
  3. Name your bucket and then click **Next** through all creation screens accepting default values
  3. Create a new folder inside this bucket to hold daily GA transaction .csv files

***



## Redshift

The following describes the process for creating a new Redshift cluster with proper accessibility and permission. Any or all of the following steps can be skipped if already created/configured.

### Setup Instructions:

  1. Create a Security Group for the Redshift Cluster so that it can be accessed within it's VPC by Glue.
      1. Navigate to the Security Group dashboard: https://console.aws.amazon.com/ec2home#SecurityGroups:sort=groupId
      2. Click **Create Security Group**
      3. Security group name: `redshift-cluster`
      4. Description: `only allow TCP connections from other services in the VPC`
      5. Select the VPC where redshift cluster is, or will be, located > *Create*
      6. In the list of Security Groups, click the security group you just created, and add the following **Inbound* and *Outbound* rules:
      7. **Inbound** rule:
          - **Type**: `All TCP`
          - **Protocol**: `TCP`
          - **Port Range**: `0-65535`
          - **Source**: start typing `redshift-cluster` and click the Security Group ID that appears
      8. **Outbound** rule:
          - **Type**: `All Traffic`
          - **Protocol**: `All`
          - **Port Range**: `All`
          - **Destination**: choose `Anywhere` from dropdown



  2. Create a Redshift Cluster
      1. Navigate to the Redshift dashboard: https://console.aws.amazon.com/redshift/home
      2. Click **Clusters** in the left sidebar
      3. Click **Launch cluster** button
      4. Fill in your preference of cluster and database details > *Continue*
      5. Select a **Node type**, **Cluster type** and **Number of compute nodes**. For a small test cluster, select:
          - `dc2.large`
          - `Single Node`
          - `1`
      6. Select the VPC and Security Group from Step #1 > *Continue*
      7. Review details > *Launch cluster*


***

## Glue

Glue is an ETL service whos main components are crawlers and jobs. Crawlers **crawl** data stores (such as S3), automatically determine the schema of contained files, and then organize data into database tables using that schmea. Glue Jobs take in the database tables created by crawlers, transform the data, and load that transformed data into a target.

### Setup Instructions:
  1. Navigate to the Glue dashboard: https://console.aws.amazon.com/glue/home

  2. Create a Glue crawler
      1. Click **Crawlers** in the left sidebar and then click **Add crawler**
      2. Enter a crawler name > *Next*
      3. Select **Data stores** > *Next*
      4. Select **S3** as your data store, then click the folder icon next to the **Include path**  field, and select the bucket and/or folder containing the data you want to crawl and include in your Glue job > *Next*
      5. Select **No** > *Next*
      6. Choose **Create an IAM role** and type `import` (full role name: `AWSGlueServiceRole-import`) > *Next*
      7. Select **Run on demand** as the Frequency > *Next*
      8. Click **Add database**, name it, and click **Create** > *Next*
      9. Confirm details > *Finish*
      10. Run your crawler by clicking **Run It Now** in popup or highlighting your crawler in the crawler dashboard and clicking **Run crawler** button.
      11. Wait for your crawler to run. When it is done it will return to the **Ready** state
      12. Make sure your crawler interpreted your file schema properly. Click **Tables** under **Databases** in the left sidebar, click the entry, and observe the **Schema** section


  3. Create a Glue job
      1. Click **Jobs** in the sidebar > click **Add job**
      2. Make the following modifications to the **Configure the job properties** screen:
          - Enter a job name
          - select IAM role `AWSGlueServiceRole-import`
          - under **Advanced properties** enable **Job Bookmark**
          - (optional) adjust location of **S3 path where the script is stored** and **Temporary directory**. Do not put these files in the same location as the location where you have designated your Glue crawler to crawl.
          - click *Next*
      3. Select a data store > **Next**
      4. Select **Change schema** > **Next**
      5. Select **Create tables in your data target** > select **JDBC** data store > click **Add connection**.
      6. In the **Add connection** popup, input details for the redshift cluster > *Next*
      7. (optional) prepare your auto-generated transform script in the interface by renaming, reordering, adding, or removing destination columns > *Next*
      7. Click **Save** at the top of the screen
      8. When you are ready to kick off your job (which will write your crawled table to Redshift) return to the **Jobs** dashboard > click your job > click **Action** button > click **Run Job**


  4. Create a Glue workflow
      1. Click **Workflows** in the left sidebar and then click **Add workflow**
      2. Enter a workflow name > *Add workflow*
      3. If you aren't already there, return to the **Workflows** dashboard > select your workflow > click **Graph** tab in the workflow edit drawer
      4. Click **Add trigger**
      5. Click **Add new** tab
      6. Name trigger `start workflow` > change **Trigger type** to `On demand` > *Add*
      7. Click **Add node** on Workflow graph > click **Crawlers** tab > select your crawler > *Add*
      8. Click the crawler node that was just created > click **Add trigger**
      9. Click **Add new** tab > name your trigger `crawler finished - start job` > select `Event` and `Start after All watched events` > *Save*
      10. Click on the trigger node you just added > click the **Action** button on above the graph > click **Add jobs/crawlers to watch**
      11. Click **Crawlers** tab > select your crawler, make sure **Crawler event to watch** is `SUCCEEDED` > *Add*
      12. Click **Add node** > click **Jobs** tab > select your job > **Add**
      13. When you are ready to kick off your workflow (which will crawl your S3 data store and then write your crawled table to Redshift when it succeeds), return to the **Workflows** dashboard > click your workflow, click **Actions** > click **Run**


***




## Lambda

### Create a Lambda function that runs a Glue Workflow when a file is uploaded to S3

### Setup Instructions:
  1. Navigate to Lambda dashbard: https://console.aws.amazon.com/lambda/home
  2. Click **Create function**
  3. Name function `start-glue-workflow-on-s3-upload` > select `Python 3.6` runtime > expand **Permissions** dropdown and select `lambda_lambda-s3-glue` > *Next*
  4. In the python code editor, paste in the following code inserting your {{GLUE WORKFLOW NAME}} between the quotations:
  ```
  import boto3
  client = boto3.client('glue')

  def lambda_handler(event, context):
    response = client.start_workflow_run(Name = '{{GLUE WORKFLOW NAME}}')
  ```

  5. Create your Lambda function trigger
      1. In the function designer, click **+ Add trigger**
      2. Select `S3`
      3. Select the S3 bucket for which you want the trigger to fire
      4. Under Event type, choose `All object create events`
      5. (optional) Add a Prefix if you want to limit the firing of the trigger to only files in specific folders in a bucket
      6. (optional) Add a Suffix if you want to limit the firing of the trigger to only specific files types
      7. Click **Add**
      8. Return to the Lambda function configuration screen > click "Save" on the top right

  6. Test your Lambda function (which should run the specified Glue workflow) by navigating to S3 and upload/copy/paste a file that satisfies the function trigger criteria. If everything is configured properly, you should see your crawler running in Glue!
