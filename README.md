### Preparation on AWS
Make an S3 bucket, in our case `chapter-platform-iceberg`.
Make a role, in our case `chapter-platform-iceberg`, which has a policy `allow-s3-platform-iceberg` that has access to the S3 bucket.
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

### Create storage integration -- TODO: don't think you need this
```sql
CREATE STORAGE INTEGRATION IF NOT EXISTS
	chapter_platform_integration
	TYPE = EXTERNAL_STAGE
	STORAGE_PROVIDER = 'S3'
	ENABLED = TRUE
	STORAGE_ALLOWED_LOCATIONS = ('s3://bucket_path/')
	STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::snowflake_arn_role';
```

### Create external volume
For the STORAGE_AWS_ROLE_ARN field, provide the ARN for the Snowflake IAM role.
For the STORAGE_ALLOWED_LOCATIONS field, provide your S3 bucket path.
This ensures that Snowflake is properly configured to interact with your S3 bucket.

```sql
CREATE OR REPLACE EXTERNAL VOLUME chapter_platform_volume
  STORAGE_LOCATIONS =
  	(
    	(
        	NAME = 'chapter_platform_volume'
        	STORAGE_PROVIDER = 'S3'
        	STORAGE_BASE_URL = 's3://chapter-platform-iceberg/icebergs'
        	STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::xxxxxxxxxxxx:role/chapter-platform-snowflake'
    	)
  	);
```

### Trust relationship
First, grab the IAM user created in your external volume. You will need it when updating the S3 policy.
‘’’sql
DESCRIBE EXTERNAL VOLUME chapter_platform_volume; -- grab the IAM user from here
‘’’

You need to create a trust relationship so that Snowflake's IAM user can assume a role that has permissions to access the S3 bucket. This involves updating the IAM role with the following policy:
```json
{
	"Version": "2012-10-17",
	"Statement": [
    	{
        	"Effect": "Allow",
        	"Principal": {
            	"AWS": "arn:aws:iam::xxxxxxxxxxxxx"
        	},
        	"Action": "sts:AssumeRole",
        	"Condition": {}
    	},
    	{
        	"Effect": "Allow",
        	"Principal": {
            	"AWS": "arn:aws:iam::the_snowflake_user_from_describing_the_external_volume"
        	},
        	"Action": "sts:AssumeRole",
        	"Condition": {
            	"StringEquals": {
                	"sts:ExternalId": [
                    	"XXXXXXXX_SFCRole=xxxxxxxxxxxxxxxxxx",
                    	"XXXXXXXX_SFCRole=xxxxxxxxxxxxxxxxxx"
                	]
            	}
        	}

```
We can finally create native iceberg tables in snowflake and you can find your parquet files in the S3 bucket.

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

```sql
select * from iceberg_scan(‘s3://chapter-platform-iceberg/icebergs/line_item’);
```

