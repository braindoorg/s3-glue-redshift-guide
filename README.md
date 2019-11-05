# Daily GA Transaction Data To Redshift: AWS Setup Guide

## Primary Services Used:
- IAM
- S3
- Glue
- Redshift
- Lambda

***

## IAM

### Purpose: creating a custom IAM role is necessary so that the glue-workflow-initiating lambda function has permissions to perform the following actions:
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

### Description: S3 will be used as the primary data store for Glue ETL jobs. It will also be used to store temporary, auto-generated files from different Amazon services as they run.

### Setup Instructions:
  1. Navigate to the S3 dashboard: https://s3.console.aws.amazon.com/s3/home
  2. Click **+ Create bucket**
  3. Name your bucket and then click **Next** through all creation screens accepting default values
  3. Create a new folder inside this bucket to hold daily GA transaction .csv files

***



## Redshift

### Description: The following describes the process for creating a new Redshift cluster with proper accessibility and permission. Any or all of the following steps can be skipped if already created/configured.

### Setup Instructions:

  1. Create a Security Group for Redshift Cluster
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
      5. Select a **Node type**, **Cluster type** and **Number of compute nodes**. For the smallest cluster, select:
          - dc2.large
          - Single Node
          - 1
      6. Select the VPC and Security Group from Step #1 > *Continue*
      7. Review details > *Launch cluster*



  3. Assign Proper security group to Redshift Cluster
      1. Go to AWS Redshift dashboard and click on "Clusters"
      2. Click the cluster checkbox, click "Cluster" button, then click "Modify Cluster"
      3. Select "redshift-cluster" in VPC security groups field and click "Modify"


***

## Glue

### Description: TODO

### Setup Instructions:
  1. Log into AWS account with permissions to view and create AWS Glue componenets and navigate to the AWS Glue dashbard

  2. Create a Glue crawler
      1. Click "Crawlers" in the left sidebar and then click "Add crawler"
      2. Enter name "ug_ga-transactions-csv-daily-crawler" and click "Next
      3. Make sure "Data stores" is selected and click "Next"
      4. Choose "S3" as your data store, click the "folder icon" next to the "Include path" input field, and select the S3 bucket where daily ga transaction csv files are stored
      5. Select "No" and click "Next"
      6. Choose "Create an IAM role", type "import" (Creating role with name AWSGlueServiceRole-import), then click "Next"
      7. Make sure "Run on demand" is selected as the Frequency and click "Next"
      8. Click "Add database", name it "ug_ga-transactions-csv-daily-db", and click "Create". When the modal dissapears click "Next"
      9. Confirm crawler setup information and click "Finish"


  3. Create a Glue job
      1. Click "Jobs" in the left sidebar and then click "Add job"
      2. Enter name "ug_ga-transactions-csv-daily-job", select the IAM role "AWSGlueServiceRole-import", and click "Next"
      3. Select the only data source listed, which should have Database "ug_transactions-csv-daily-db" and click "Next"
      4. Make sure "Change schema" is selected and click "Next"
      5. Select "Create tables in your data target", Select "JDBC" data store, then click "Add connection". Add the information for the redshift cluseter and database where you want to add this job's database tables to be written. Type in the database name, then click "Next".
      6. Rename all columns on the right hand side of the mapping, removing the "ga:" prefix, then click "Save job and edit script".
      7. Click "Save" at the top of the screen


  4. Create a Glue workflow
      1. Click "Workflows" in the left sidebar and then click "Add workflow"
      2. Enter name "ug_ga-transactions-csv-daily-workflow"
      3. Go back to the "Workflows" dashbpoard, highlight your new workflow row, and expand the "Graph" section at the bottom of the screen
      4. Click "Add trigger"
      5. Click "Add new" tab
      6. Name trigger "start workflow", switch "Trigger type" to "On demand" and click "Add"
      7. Click "Add node" on Workflow graph, click "Crawlers" tab, select your crawler, and click "Add"
      8. Click the crawler node that was just created and then click "Add trigger"
      9. Click "Add new" tab, name your trigger "crawler finished - start job", select "Event" and "Start after All watched event", then click Save
      10. Click on the trigger node you just added, click the "Action" button on above the graph, then click "Add jobs/crawlers t owatch"
      11. Click "Crawlers" tab, select your crawler, make sure "Crawler event to watch" is "SUCCEEDED", then click Add
      12. Click "Add node", click "Jobs" tab, select your job, then click "Add"


***




## Lambda

### Description: create a Lambda function that will start the Glue Workflow when the daily ga transactions .csv is uploaded to S3.

### Setup Instructions:
  1. Navigate to Lambda dashbard of AWS
  2. Click "Create function"
  3. Name function `start-glue-workflow-on-daily-upload`, select `Python 3.6` runtime, then expand "Permissions" dropdown and select the role created in the IAM section of this document, `lambda_lambda-s3-glue`. Click Next.
  4. In code editor, paste in the following code:
  ```
  import boto3
  client = boto3.client('glue')

  def lambda_handler(event, context):
    response = client.start_workflow_run(Name = 'ug_ga-transactions-csv-daily-workflow')
  ```

  6. Create your Lambda function trigger
      1. In the Designer, click "+ Add trigger"
      2. Select S3
      3. Select the S3 bucket
      4. Under Event type, choose "All object create events"
      5. Add a Prefix: (optional) if .csv files are not at the root of the bucket but in folders, add the folder path here with a trailing slash
      6. Add a Suffix: .csv
