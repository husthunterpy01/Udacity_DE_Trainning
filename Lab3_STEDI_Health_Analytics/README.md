# STEDI Human Balance Analysis project

# Overview

In this project, I will work on building a lakehouse solution for sensor data used to train for a machine learning model based on the **STEDI Step Trainer**data

Steps to follow:
1. Download customer, accelerometer, and step trainer data from the zip file.
2. Fix the formatting error and incomplete files. We call this the complete & cleaned data.
3. Upload the complete & cleaned data to AWS Glue.
4. Reduce the number of data points but do it smartly. Keep only relevant data.
5. Redo the project with the new data.
6. Record issues met and their solutions.


# Highlights
## Data
Original data:
In this project, we will work with three main data types, including 
- Customer Records:
  * serialnumber
  * sharewithpublicasofdate
  * birthday
  * registrationdate
  * sharewithresearchasofdate
  * customername
  * email
  * lastupdatedate
  * phone
  * sharewithfriendsasofdate
- Step Trainer Records:
  * sensorReadingTime
  * serialNumber
  * distanceFromObject
- Accelerometer Records:
  * timeStamp
  * user
  * x
  * y 
  * z
  
Complete & Cleaned data:

- Landing
  * Customer: 956
  * Accelerometer: 81273
Step Trainer: 28680
- Trusted
  * Customer: 482
  * Accelerometer: 40981
Step Trainer: 14460
- Curated
  * Customer: 482
  * Machine Learning: 43681

- Their relationship is presented within this ERD:
![Header](./ERD_Lakehouse.png)

## Project flowchart
This project is conducted through AWS Services, such as AWS Athena, AWS Glue and and AWS S3:
![Header](./flowchart.jpg)

## Files Directories
```plaintext
Lab3_STEDI_Health_Analytics
├── Data_Checked
    ├── accelerometer
         ├── accelerometer_landing.png
         ├── accelerometer_trusted.png
    ├── customer
         ├── customer_landing.png
         ├── customer_trusted.png
         ├── customers_curated.png
         ├── customer_landing_check_null.png
         ├── customer_trusted_check_null.png
    ├── step_trainer
         ├── step_trainer_landing.png
         ├── step_trainer_trusted.png
    ├── ml_curated.png
├── Spark Glue Job
         ├── accelerometer_landing_to_trusted.py
         ├── customer_landing_to_trusted.py
         ├── customer_trusted_to_curated.py
         ├── machine_learning_curated.py
         ├── steptrainer_landing_to_trusted.py
├── Table_DLL
         ├── accelerometer_trusted.sql
         ├── acclerometer_landing.sql
         ├── customer_curated.sql
         ├── customer_landing.sql
         ├── customer_trusted.sql
         ├── ml_curated.sql
         ├── step_trainer_landing.sql
         ├── step_trainer_trusted.sql
├── readme_images
         ├── cust_curated_issue-1.png
         ├── cust_curated_issue-2.png
         ├── cust_curated_issue-3.png
         ├── cust_curated_issue-4.png
         ├── dropfields_dont_work-1.png
         ├── dropfields_dont_work-2.png
         ├── dropfields_dont_work-3.png
         ├── dropfields_dont_work-6.png
         ├── incorrect-customer_landing_to_trusted.png
├── README.md
└── starter
         ├── accelerometer/landing
         ├── customer/landing
         ├── step_trainer/landing
```

- The ```data_check directory``` will show the result of each tables after being process through AWS Glue and presented in AWS Athena
- The ```Spark Glue Job``` indicates the script for Glue Data processing with the help of Spark and SparkSQL
- ```Table_DLL``` notes the DLL of the tables formed by the join and by default format, which situates in the S3 storage
- ```readme_images``` demonstrates some configuration using AWS Glue visual to handle with the base data to convert into the state of landing, trusted and curated
- ```starter``` defines the base data for this lab, which will later be copied to S3 storage

# Steps of execution
## PreSetup for AWS Environment
### S3 Bucket Configuration
- Creating S3 bucket, using the command aws s3 mb ```s3://your-bucket```
- Search for S3 Gateway endpoint with the command ```aws ec2 describe-vpcs```, with the result just like the following:
  ```
  {
    "Vpcs": [
        {
            "CidrBlock": "172.31.0.0/16",
            "DhcpOptionsId": "dopt-756f580c",
            "State": "available",
            "VpcId": "vpc-7385c60b",
            "OwnerId": "863507759259",
            "InstanceTenancy": "default",
            "CidrBlockAssociationSet": [
                {
                    "AssociationId": "vpc-cidr-assoc-664c0c0c",
                    "CidrBlock": "172.31.0.0/16",
                    "CidrBlockState": {
                        "State": "associated"
                    }
                }
            ],
            "IsDefault": true
        }
    ]
  ```
= Search the routing table with the command ```aws ec2 describe-route-tables``` by this format:
```{
    "RouteTables": [

        {
      . . .
            "PropagatingVgws": [],
            "RouteTableId": "rtb-bc5aabc1",
            "Routes": [
                {
                    "DestinationCidrBlock": "172.31.0.0/16",
                    "GatewayId": "local",
                    "Origin": "CreateRouteTable",
                    "State": "active"
                }
            ],
            "Tags": [],
            "VpcId": "vpc-7385c60b",
            "OwnerId": "863507759259"
        }
    ]
```

- Acquire the routing-table and vpc-id values, and replace it in this following command to create S3 Gateway Endpoint
  ```\aws ec2 create-vpc-endpoint --vpc-id _______ --service-name com.amazonaws.us-east-1.s3 --route-table-ids _______```

### Create S3 IAM Role
Enter the following command in the AWS Cli to authorize the S3 to interact with the Athena and Glue service
- Enter the new role: 
```
aws iam create-role --role-name my-glue-service-role --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "glue.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]}'
```
- Grant Glue Priviliges:
  ```
  aws iam put-role-policy --role-name my-glue-service-role --policy-name GlueAccess --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "glue:*",
                "s3:GetBucketLocation",
                "s3:ListBucket",
                "s3:ListAllMyBuckets",
                "s3:GetBucketAcl",
                "ec2:DescribeVpcEndpoints",
                "ec2:DescribeRouteTables",
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSubnets",
                "ec2:DescribeVpcAttribute",
                "iam:ListRolePolicies",
                "iam:GetRole",
                "iam:GetRolePolicy",
                "cloudwatch:PutMetricData"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:CreateBucket",
                "s3:PutBucketPublicAccessBlock"
            ],
            "Resource": [
                "arn:aws:s3:::aws-glue-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::aws-glue-*/*",
                "arn:aws:s3:::*/*aws-glue-*/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::crawler-public*",
                "arn:aws:s3:::aws-glue-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:AssociateKmsKey"
            ],
            "Resource": [
                "arn:aws:logs:*:*:/aws-glue/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags",
                "ec2:DeleteTags"
            ],
            "Condition": {
                "ForAllValues:StringEquals": {
                    "aws:TagKeys": [
                        "aws-glue-service-resource"
                    ]
                }
            },
            "Resource": [
                "arn:aws:ec2:*:*:network-interface/*",
                "arn:aws:ec2:*:*:security-group/*",
                "arn:aws:ec2:*:*:instance/*"
            ]
        }
    ]}'
  ```

- Glue Policy
```
aws iam put-role-policy --role-name my-glue-service-role --policy-name GlueAccess --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "glue:*",
                "s3:GetBucketLocation",
                "s3:ListBucket",
                "s3:ListAllMyBuckets",
                "s3:GetBucketAcl",
                "ec2:DescribeVpcEndpoints",
                "ec2:DescribeRouteTables",
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSubnets",
                "ec2:DescribeVpcAttribute",
                "iam:ListRolePolicies",
                "iam:GetRole",
                "iam:GetRolePolicy",
                "cloudwatch:PutMetricData"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:CreateBucket",
                "s3:PutBucketPublicAccessBlock"
            ],
            "Resource": [
                "arn:aws:s3:::aws-glue-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::aws-glue-*/*",
                "arn:aws:s3:::*/*aws-glue-*/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::crawler-public*",
                "arn:aws:s3:::aws-glue-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:AssociateKmsKey"
            ],
            "Resource": [
                "arn:aws:logs:*:*:/aws-glue/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags",
                "ec2:DeleteTags"
            ],
            "Condition": {
                "ForAllValues:StringEquals": {
                    "aws:TagKeys": [
                        "aws-glue-service-resource"
                    ]
                }
            },
            "Resource": [
                "arn:aws:ec2:*:*:network-interface/*",
                "arn:aws:ec2:*:*:security-group/*",
                "arn:aws:ec2:*:*:instance/*"
            ]
        }
    ]
}'
```
## Extract and load data
- We will work on three types data, known as **customer**, **step_trainer** and **accelerometer**, in which we will have to copy into the s3 storage
  To do that, please clone this repo and then point to the directory of the data, and then enter this command
   ``` aws s3 cp ./project/starter/customer/landing/customer-1691348231425.json s3://_______/customer/landing/```
- Then please visit AWS Glue, and import or code by the Spark Glue Job file, or just drag-and-drop the box and declare the relevant information of the data to export the desire data after processing
- We can create the Table after processing in AWS Athena, and then declare the directory where the output data lying in S3 to pour the data into the table. After that, please check the table by implementing the SQL Query on Athena.
  Here is a glance at the landing table
  - customer_landing
    ![Header](./Data_Checked/customer/customer_landing.png)
  - acceleromter_landing
    ![Header](./Data_Checked/accelerometer/accelerometer_landing.png)
  - step_trainer_landing
    ![Header](./Data_Checked/step_trainer/step_trainer_landing.png)
- Clean the data and categorize the data into the 3 types above with the following requirements:
1. Sanitize the Customer data from the Website (Landing Zone) and only store the Customer Records who agreed to share their data for research purposes (Trusted Zone) - creating a Glue Table called **customer_trusted** - referring to [customer_landing_to_trusted.py](./Spark%20Glue%20Job/customer_landing_to_trusted.py)
![Header](./Data_Checked/customer/customer_trusted.png)

2. Sanitize the Accelerometer data from the Mobile App (Landing Zone) - and only store Accelerometer Readings from customers who agreed to share their data for research purposes (Trusted Zone) - creating a Glue Table * * called **accelerometer_trusted**. [accelerometer_landing_to_trusted.py](./Spark%20Glue%20Job/accelerometer_landing_to_trusted.py)
![Header](./Data_Checked/accelerometer/accelerometer_trusted.png)
You need to verify your Glue job is successful and only contains Customer Records from people who agreed to share their data. Query your Glue customer_trusted table with Athena and take a screenshot of the data. Name the screenshot customer_trusted(.png,.jpeg, etc.).

- Data Scientists have discovered a data quality issue with the Customer Data. The serial number should be a unique identifier for the STEDI Step Trainer they purchased. However, there was a defect in the fulfillment website, and it used the same 30 serial numbers over and over again for millions of customers! Most customers have not received their Step Trainers yet, but those who have, are submitting Step Trainer data over the IoT network (Landing Zone). The data from the Step Trainer Records has the correct serial numbers.

The Data Science team would like you to write a Glue job that does the following:

1. Sanitize the Customer data **(Trusted Zone)** and create a Glue Table **(Curated Zone)** that only includes customers who have accelerometer data and have agreed to share their data for research called customers_curated. [customer_trusted_to_curated.py](./Spark%20Glue%20Job/customer_trusted_to_curated.py)
![Header](./Data_Checked/customer/customer_curated.png)
Finally, you need to create two Glue Studio jobs that do the following tasks:

1. Read the Step Trainer IoT data stream (S3) and populate a Trusted Zone Glue Table called **step_trainer_trusted** [steptrainer_landing_to_trusted.py](./Spark%20Glue%20Job/steptrainer_landing_to_trusted.py) that contains the Step Trainer Records data for customers who have accelerometer data and have agreed to share their data for research (**customers_curated**).
![Header](./Data_Checked/step_trainer/step_trainer_trusted.png)
2. Create an aggregated table that has each of the Step Trainer Readings, and the associated accelerometer reading data for the same timestamp, but only for customers who have agreed to share their data, and make a glue table called **machine_learning_curated**. [ml_curated.py](./Spark%20Glue%20Job/ML_curated.py)
![Header](./Data_Checked/ml_curated.png)
# Queries
## All connected rows and sanitized

```
SELECT COUNT(*)

FROM "customer_cleaned_landing" cl

    JOIN "accelerometer_cleaned_landing" al ON cl.email = al.user

    JOIN "step_trainer_cleaned_landing" sl ON cl.serialnumber = sl.serialnumber AND al.timestamp = sl.sensorreadingtime

WHERE cl.sharewithresearchasofdate IS NOT NULL;
```

- There are currently 2,043,198 rows.
- Run time in Athena: 15.173 sec
- Data scanned 316.20 MB

## How many distinct emails are there?

```
SELECT COUNT(DISTINCT email) FROM "customer_cleaned_landing";
```

There are only 957 distinct emails.


## Are there duplicates in step trainer data (duplicated `sensorreadingtime` and `serialnumber` pairs, that is)?

```
SELECT sensorreadingtime, serialnumber, COUNT(*)
FROM step_trainer_cleaned_landing
GROUP BY sensorreadingtime, serialnumber
HAVING COUNT(*) > 1;
```

## Reduced data rows

* Get unique customers by emails with the earliest registrationDate.
* Get relevant accelerometer and step trainer data of those customers.

```
WITH
    cl_distinct_emails AS (
        SELECT *, row_number() OVER (PARTITION BY email ORDER BY email, registrationDate DESC) AS row_num
        FROM customer_cleaned_landing

    )
SELECT DISTINCT *
FROM cl_distinct_emails cl
    JOIN accelerometer_cleaned_landing al
        ON cl.email = al.user AND cl.row_num = 1
    JOIN step_trainer_cleaned_landing sl
        ON cl.serialnumber = sl.serialnumber
            AND al.timestamp = sl.sensorreadingtime;
```

Results: 81,273 rows

# Some precautions:
- I would advise to use the Spark SQL for joinning table, as this would handle SQL Query better than Glue Studio
- When converting the output data, you should include all the files into a single file, for my case, instead as of having many .snappy files I would rather use a single .json file so as to query the data from Athena easier and avoiding the risk of data joinning mismatch or errors for later ETL process.
- For the customer_curated data, it's the best to drop duplicate data to get the desired records, if not we will get over 40k records with duplicate data.

