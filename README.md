# Quack Quack Ka-Ching
Playbook for reading snowflake-managed iceberg tables with DuckDB.
Blog post at https://medium.com/datamindedbe

### Preparation on AWS
Make an S3 bucket, in our case `chapter-platform-iceberg`.
Make an IAM policy, in our case `allow-s3-platform-iceberg` that has access to that S3 bucket.
```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "AllowPlatformIcebergBucket",
			"Effect": "Allow",
			"Action": "s3:*",
			"Resource": [
				"arn:aws:s3:::chapter-platform-iceberg",
				"arn:aws:s3:::chapter-platform-iceberg/*"
			]
		}
	]
}
```
Make a role, in our case `chapter-platform-iceberg`, and attach the above policy to it.
We'll need to set a trust policy for this role later, so that Snowflake can assume it.

### Create external volume
For the `STORAGE_AWS_ROLE_ARN` field, provide the ARN of the IAM role that we just made.
For the `STORAGE_ALLOWED_LOCATIONS` field, provide your S3 bucket path.
```sql
CREATE OR REPLACE EXTERNAL VOLUME chapter_platform_volume
  STORAGE_LOCATIONS =
  	(
    	(
        	NAME = 'chapter_platform_volume'
        	STORAGE_PROVIDER = 'S3'
        	STORAGE_BASE_URL = 's3://chapter-platform-iceberg/icebergs'
        	STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::xxxxxxxxxxxx:role/chapter-platform-iceberg'
    	)
  	);
```
This ensures that Snowflake is properly configured to interact with your S3 bucket.

### Trust relationship
First, grab the arn of the IAM user in your external volume.
This IAM user is one that Snowflake created and that lives in Snowflake's AWS account.
You will need it to update the trust policy of your AWS role, along with the `EXTERNAL_ID`.
This is because for different external volumes, the same IAM user is used with different `EXTERNAL_IDs.
‘’’sql
DESCRIBE EXTERNAL VOLUME chapter_platform_volume;
‘’’
One of the `property_value` fields - once formatted - will look something like this:
```json
{
    "NAME":"chapter_platform_volume",
    "STORAGE_PROVIDER":"S3",
    "STORAGE_BASE_URL":"s3://chapter-platform-iceberg/icebergs",
    "STORAGE_ALLOWED_LOCATIONS":["s3://chapter-platform-iceberg/icebergs*"],
    "STORAGE_REGION":"eu-west-1",
    "PRIVILEGES_VERIFIED":true,
    "STORAGE_AWS_ROLE_ARN":"arn:aws:iam::xxxxxxxxxxxx:role/chapter-platform-snowflake",
    "STORAGE_AWS_IAM_USER_ARN":"arn:aws:iam::xxxxxxxxxxxx:user/xxxxxxxxxxxx",
    "STORAGE_AWS_EXTERNAL_ID":"XXXXXXXX_SFCRole=xxxxxxxxxxxxxxxxxx",
    "ENCRYPTION_TYPE":"NONE",
    "ENCRYPTION_KMS_KEY_ID":""}
```
Make sure you copy this `STORAGE_AWS_IAM_USER_ARN` and `STORAGE_AWS_EXTERNAL_ID`.

Now, on AWS side, you need to create a trust relationship so that Snowflake's IAM user can assume a role that has permissions to access the S3 bucket.
This involves updating the IAM role to have the following trust policy:
```json
{
	"Version": "2012-10-17",
	"Statement": [
    	{
        	"Effect": "Allow",
        	"Principal": {
            	"AWS": "arn:aws:iam::xxxxxxxxxxxx:user/xxxxxxxxxxxx"
        	},
        	"Action": "sts:AssumeRole",
        	"Condition": {
            	"StringEquals": {
                	"sts:ExternalId": [
                    	"XXXXXXXX_SFCRole=xxxxxxxxxxxxxxxxxx"
                	]
            	}
        	}
        }
    ]
}
```

We can finally create native iceberg tables in Snowflake and you can find your parquet files in the S3 bucket.
```sql
CREATE OR REPLACE ICEBERG TABLE line_item
BASE_LOCATION='line_item'
EXTERNAL_VOLUME=chapter_platform_volume
CATALOG=SNOWFLAKE
AS (SELECT * FROM SNOWFLAKE_SAMPLE_DATA.TPCH_SF1.LINEITEM);
```

### DuckDB
To begin, ensure you have the necessary dependencies installed and configured in DuckDB.

```sql
install iceberg;
load iceberg;
install aws;
load aws;
install httpfs;
load httpfs;
call load_aws_credentials();
```

Then, you can start querying your iceberg table.
```sql
select * from iceberg_scan(‘s3://chapter-platform-iceberg/icebergs/line_item’);
```

Enjoy!
