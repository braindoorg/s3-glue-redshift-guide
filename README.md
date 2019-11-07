# Creating an Service Pipeline in AWS to Import and Transform Data from S3 into Redshift

The following instructions assume you are logged into AWS with an account full permission to list, modify, and create services. It also assumes you have an AWS region selected where you want to create these services.

## Main Services Involved:
- IAM
- VPC and VPC Security Groups
- S3
- Glue
- Redshift
- Lambda


***


## IAM

Create a role for our Lamda function so it has permissions to:
- detect a change to S3
- execute a python script that runs a Glue workflow
- write execution/error logs to CloudWatch and CloudTrail

### Setup Instructions:
  1. Navigate to the IAM dashboard: https://console.aws.amazon.com/iam/home
  2. Click **Roles** in the sidebar
  3. Click **Create role**
  4. Under **Choose the service that will use this role**, click **Lambda** > *Next: Permissions*
  5. Search for, and check two policies: `AWSLambdaBasicExecutionRole` and `AWSGlueServiceRole` > *Next: Tags*
  6. (optional) tag your Role with a Key/Value pair > *Next: Review:*
  6. Add Role name `lambda_lambda-s3-glue` > *Create role*


***

## VPC and VPC Security Groups

Add a VPC endpoint and VPC Security group so that services can communicate with one another in the VPC.

### Setup Instructions:
  1. Create a VPC endpoint
      1. Navigate to the VPC Endpoints dashboard: https://console.aws.amazon.com/vpc/home#Endpoints:sort=vpcEndpointId
      2. Click **Create Endpoint**
      3. Select **Find service by name** and enter `com.amazonaws.{{AWS REGION}}.s3`, substituting your region for {{AWS REGION}} (ex, `com.amazonaws.us-east-1.s3`)
      4. Click **Verify** button to confirm that the service name was found
      5. In the **Configure route tables** section, check the box next to the route table row in the table
      6. Click **Create endpoint**

  2. Create a Security Group that allows resources to accept inbound connections from other services in the same VPC security group. This Security Group will be used exclusively for our Redshift cluster.
      1. Navigate to the Security Group dashboard: https://console.aws.amazon.com/ec2/home#SecurityGroups:
      2. Click **Create Security Group**
      3. Enter **Security group name**: `redshift-cluster`
      4. Enter **Description**: `only allow TCP connections from other services in the VPC`
      5. Select the VPC where other services are/will-be located > *Create*
      6. Select the `redshift-cluser` security group. In the drawer add the following **Inbound** rule:
      7. **Inbound** rule:
          - **Type**: `All TCP`
          - **Protocol**: `TCP`
          - **Port Range**: `0-65535`
          - **Source**: start typing `redshift-cluster` and click the Security Group ID that appears



## S3

S3 will be used as the primary data store for data to be processed and uploaded to Redshift. It will also be used to store permenant scripts and temporary files created by Glue.

### Setup Instructions:
  1. Navigate to the S3 dashboard: https://s3.console.aws.amazon.com/s3/home
  2. Click **+ Create bucket**
  3. Enter a **Bucket name** and then keep clicking **Next** through all creation screens accepting default values
  4. Once the bucket is created, upload one file appropriate for your project for testing purposes.

Note: You may choose to store files within folders instead of uploading all to the root of the S3 bucket. If you do, you will need to slightly modify the S3 paths used for the Glue crawler and Lambda function S3 trigger.

***


## Redshift

The following describes the process for creating a new Redshift cluster with proper accessibility and permission. Any or all of the following steps can be skipped if already created/configured.

### Setup Instructions:
  1. Create a Redshift Cluster
      1. Navigate to the Redshift dashboard: https://console.aws.amazon.com/redshift/home
      2. Click **Clusters** in the left sidebar
      3. Click **Launch cluster**
      4. Add a cluster name and database credentials > *Continue*
      5. Select a **Node type**, **Cluster type** and **Number of compute nodes**. For a small test cluster, select:
          - `dc2.large`
          - `Single Node`
          - `1`
      6. Select the appropriate VPC and `redshift-cluser` **VPC security group** > *Continue*
      7. Review details > *Launch cluster*


Note: Redshift clusters are fairly expensive; if you created a new Redshift Cluster for the purposes of testing this workflow, remember to delete it and create a snapshot as soon as you are done using it.


***


## Glue

Glue is an ETL service whos main components are crawlers and jobs. Crawlers *crawl* data stores (such as S3), determine the schema of contained files, and then create tables with that schema and file data. Glue Jobs take in database tables created by crawlers, transform the data, and load that transformed data into a target.

### Setup Instructions:
  1. Navigate to the Glue dashboard: https://console.aws.amazon.com/glue/home


  2. Create a Glue crawler
      1. Click **Crawlers** in the left sidebar > click **Add crawler**
      2. Enter a **Crawler name** > *Next*
      3. Select **Data stores** > *Next*
      4. Select **S3** data store > click the folder icon next to the **Include path** > select the bucket/folder containing the data you want to crawl > *Next*
      5. Select **No** > *Next*
      6. Choose **Create an IAM role** and type `import` (full role name: **AWSGlueServiceRole-import**) > *Next*
      7. Select **Run on demand** as the **Frequency** > *Next*
      8. Click **Add database**, enter a **Database name** > **Create** > *Next*
      9. Confirm details > *Finish*
      10. Run your crawler. Click **Run It Now ?** in the popup or highlight your crawler in the crawler dashboard > **Run crawler**.
      11. Wait for your crawler to run. When it is done it will return to the **Ready** state
      12. Check to make sure your crawler interpreted your file's schema properly. Click **Tables** under **Databases** in the left sidebar, click the entry, and observe the **Schema** section

  2. Create a Connection (between Glue and your redshift cluster)
      1. Click **Connection** in the left sidebar > click **Add connection**
      2. Enter a **Connection name** > select `Amazon Redshift` as the **Connection type** > *Next*
      3. Select your redshift cluster from the **Cluster** dropdown > (optional) change the **Database name** if what auto-populated is not correct > Enter **Username** and **Password** for database created in Redshift > *Next*
      4. Review details > *Finish*

  3. Create a Glue job
      1. Click **Jobs** in the sidebar > click **Add job**
      2. Do the following on the **Configure the job properties** screen:
          - Enter a job name
          - select IAM role `AWSGlueServiceRole-import`
          - under **Advanced properties** enable **Job Bookmark**
          - (optional) adjust **S3 path where the script is stored** and **Temporary directory**. Do not put these files in the same location as where your Glue crawler crawls.
          - click *Next*
      3. Select a data source (glue crawler database/table(s)) > *Next*
      4. Select **Change schema** > *Next*
      5. Select `Create tables in your data target` > select `JDBC` **Data store** > select **Connection** added in step 2 > Enter your Redshift cluster **Database name** > *Next*
      6. (optional) prepare your python transform script by renaming, reordering, adding, or removing destination columns > *Save job and edit script*
      7. Click **Save** at the top left of the screen
      8. When you are ready to kick off your job (which will write your crawled table to Redshift) click **Run job** next to save, or return to the **Jobs** dashboard > click your job > click **Action** button > click **Run Job**. No advanced settings are necessary.


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

Create a Lambda function that runs a Glue Workflow when a file is uploaded to S3

### Setup Instructions:
  1. Navigate to Lambda dashbard: https://console.aws.amazon.com/lambda/home
  2. Click **Create function**
  3. Name function `start-glue-workflow-on-s3-upload` > select `Python 3.6` runtime > expand **Permissions** dropdown and select `lambda_lambda-s3-glue` Role > *Next*
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
      3. Select the S3 bucket for which you want the trigger to fire when objects are created in it
      4. Under Event type, choose `All object create events`
      5. (optional) Add a Prefix if you want to limit the firing of the trigger to only files in specific folders (i.e. `daily-exports/`)
      6. (optional) Add a Suffix if you want to limit the firing of the trigger to only specific files types (i.e. `.csv`)
      7. Click **Add**
      8. Return to the Lambda function configuration screen > click **Save** on the top right


  6. Test your Lambda function (which should run the specified Glue workflow) by navigating to S3 and upload/copy/paste a file that satisfies the function trigger criteria. If everything is configured properly, you should see your crawler set to the running state in Glue.
