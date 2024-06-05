# Change Data Capture (CDC) / Replication On Going pipeline with AWS and PySpark


## Requirements

The end goal of this project is to have a fully functional CDC pipeline.
The pipeline will cover two phases or modes:
1. Full Load to replicate the initial database
2. CDC / Replication on Going to process database changes


Initially, it will extract the full load - all data present inside the database - and load it in a temporary HDFS / AWS S3 bucket storage.
Then, it will keep on looking for any changes made inside the source database and compensate that changes inside the destination storage. 
The main essence is that it will create a data lake out of this database and any future changes in the database (insertion, deletion, updating, etc.).


## Architecture

![alt text](architecture/CDC.png)



As you can see in the architecture, there are a few things we need to set up the cloud infrastructure to run this project.
For this, I used some AWS cloud services such as RDS, DMS, S3, Lambda and Glue for PySpark job

1. RDS MySQL database instance for Source Endpoint
2. Temporary S3 Bucket for temporary Destination / target Endpoint
3. DMS setup with two endpoints attached: a source endpoint and a destination endpoint
4. DMS reads data from Source Endpoint RDS MySQL database instance
5. DMS writes data to the Destination Endpoint - Temp S3 bucket
6. Lambda function with a configured trigger which will be invoked when a new data file lands in the Temp S3 bucket
7. Glue PySpark job triggered by the Lambda function to process / transform the data and load / write it in the final destination S3 Bucket
8. Set up IAM roles to allow access between services: S3 - Lambda - Glue - CloudWatch
9. Set up CloudWatch to have access to Lambda and Glue PySpark job logs
10. Final S3 Bucket for the final destination - data lake


## Implementation

1. Create a RDS MySQL database instance, connect it with the MySQL workbench on the local machine and populate it with data (dump data) using SQL commands executed from the MySQL workbench (see dump file). After dumping the data in the MySQL db, run a simple SQL query to check that everything is ok. The RDS MySQL database instance will be the source endpoint for the DMS.

2. Create the temporary S3 bucket. This will be the destination or target endpoint for the DMS service, where the date will be loaded temporally to be transformed with the Glue PySpark job, before being loaded in the final S3 bucket (data lake).

3. Create DMS replication instance and the two endpoints.

4. Create DMS source endpoint (RDS -> DMS) and configure it with the RDS instance 

5. Create DMS target endpoint (DMS -> S3) and configure it with the S3 Bucket

6. Implement Lambda function and configure the triggering and invoking mechanism if any file lands inside the Temp S3 bucket. The Lambda will get the name of the new update data file that triggered it and it will pass it to the Glue PySpark for further processing the changes.

Lambda code:

```python
import json
import boto3

def lambda_handler(event, context):
    
    
    bucketName = event["Records"][0]["s3"]["bucket"]["name"]
    fileName = event["Records"][0]["s3"]["object"]["key"]
    
    print(bucketName, fileName)
        
    glue = boto3.client('glue')

    response = glue.start_job_run(
        JobName = 'glueCDC-pyspark',
        Arguments = {
            '--s3_target_path_key': fileName,
            '--s3_target_path_bucket': bucketName
        } 
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

7. Create the Glue PySpark job which will be invoked by the Lambda function to transform the data and load / write it in the final destination S3 Bucket. Use "boto3" Python library to set up communication between Lambda and Glue PySpark job 

Glue PySpark job

```python
from awsglue.utils import getResolvedOptions
import sys
from pyspark.sql.functions import when
from pyspark.sql import SparkSession

args = getResolvedOptions(sys.argv,['s3_target_path_key','s3_target_path_bucket'])
bucket = args['s3_target_path_bucket']
fileName = args['s3_target_path_key']

print(bucket, fileName)

spark = SparkSession.builder.appName("CDC").getOrCreate()
inputFilePath = f"s3a://{bucket}/{fileName}"
finalFilePath = f"s3a://cdc-output-pyspark/output"

if "LOAD" in fileName:
    fldf = spark.read.csv(inputFilePath)
    fldf = fldf.withColumnRenamed("_c0","id").withColumnRenamed("_c1","FullName").withColumnRenamed("_c2","City")
    fldf.write.mode("overwrite").csv(finalFilePath)
else:
    udf = spark.read.csv(inputFilePath)
    udf = udf.withColumnRenamed("_c0","action").withColumnRenamed("_c1","id").withColumnRenamed("_c2","FullName").withColumnRenamed("_c3","City")
    ffdf = spark.read.csv(finalFilePath)
    ffdf = ffdf.withColumnRenamed("_c0","id").withColumnRenamed("_c1","FullName").withColumnRenamed("_c2","City")
    
    for row in udf.collect(): 
      if row["action"] == 'U':
        ffdf = ffdf.withColumn("FullName", when(ffdf["id"] == row["id"], row["FullName"]).otherwise(ffdf["FullName"]))      
        ffdf = ffdf.withColumn("City", when(ffdf["id"] == row["id"], row["City"]).otherwise(ffdf["City"]))
    
      if row["action"] == 'I':
        insertedRow = [list(row)[1:]]
        columns = ['id', 'FullName', 'City']
        newdf = spark.createDataFrame(insertedRow, columns)
        ffdf = ffdf.union(newdf)
    
      if row["action"] == 'D':
        ffdf = ffdf.filter(ffdf.id != row["id"])
        
    ffdf.write.mode("overwrite").csv(finalFilePath)   
```

8. Set up IAM roles to allow access between services: S3 - Lambda - Glue - CloudWatch

9. Set up CloudWatch to have access to Lambda and Glue PySpark job logs

10. Create final S3 Bucket for the final destination for the data - data lake


## Test the CDC pipeline

### 1. Test Full Load mode

1. Dump some data in the RDS MySQL database
2. This will trigger the DMS migration service which will replicate the data (Full Load at first CDC pipeline run) and load it in the Temp S3 Bucket.
3. When the data file lands in the Temp S3 Bucket, it will trigger the Lambda function which will grab the file name and the bucket name and pass it to the Glue PySpark job and invoke it. Check the Lambda CloudWatch logs to see the results.
4. The Glue PySpark job will check if the data is the full load (initial database) or new changes in the database -> at first run we are in the Full Load scenario, so it will simply replicate the initial database in the Final S3 Bucket. Check the Glue PySpark job CloudWatch logs for the results. Also, download the data file from the final S3 bucket and compere it with the initial database, should be the same.

### 2. Test CDC / Replication On Going mode

1. Dump some data changes in the RDS MySQL database (insert, delete, update, ...).
2. This will trigger the DMS migration service which will replicate the data file with the changes made in the database and load it in the Temp S3 Bucket.
3. When the data file lands in the Temp S3 Bucket, it will trigger the Lambda function which will grab the file name and the bucket name and pass it to the Glue PySpark job and invoke it. Check the Lambda CloudWatch logs to see the results.
4. The Glue PySpark job will check if the data is the full load (initial database) or new changes in the database -> now we are in the CDC scenario, so it will process / transform the data (check the Glue code) and then append or compensate the modified data in the Final S3 Bucket data lake. Check the Glue PySpark job CloudWatch logs for the results. Also, download the data file from the final S3 bucket and compere it with the initial database, it should have all the changes.