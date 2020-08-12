+++
title = "Cloning External FTP Servers with AWS"

[taxonomies]
categories = ["Useful Airflow Patterns"]

+++

A common way to run airflow is a stating/production instance where DAGs are tested in staging and then promoted to production. You'd like the same workflows to run in staging and production, but you don't want staging workflows interfering with production(touching the same files, kicking off the same proceses, etc). You especially don't want your staging tasks messing up files from external vendors. But you also don't want to wire up a copying step after every ftp transfer (and have those possibly interfer with futher downstream dag processing). You'd like to be able to test ftp related changes in the staging env and have resonable confidence that it will work.

To acomplish these thigns we can use an event driven lambda service and AWS Transfer to transparently clone your vendor ftp files.

Assumptions - you're using AWS and S3 for your file storage.


First create set up AWS transfer and create login credentials for your external FTP sites. You can use the same s3 bucket for all of them and point each to different sub directories.

Then you want to create a lambda function an add `ObjectCreated` triggers for any source bucket (buckets your production airflow instances are copying files to).

Then we need a structure to hold our rules:

```python
# An `S3Router` will be:
# Dict[str, Tuple[str, Union[None, callable]]
# S3Router = {
#    'src_key_pattern': ('dest_bucket', 'dest_key')
# }
# where `dest_key` is a None, at which point we reuses the src_key, or
# a callable
# an example for our purposes:
S3Router = {
    ".*a_very_important_file.\d{8}.csv" : (
        "my-staging-s3-bucket",
        None
    ),
    ".*a_different_file.latest.csv": (
        "my-aws-transfer-buckket",
        lambda x: "ftp_dir/outgoing"+Path(x).name
    ),
}
```

Here - we'd like to grab the  `a_very_important_file` and move them to our staging s3 bucket without making any modifications to the s3 key. We'd also like to grab `a_different_file` and move it to our AWS Transfer bucket under the path we would have picked it up from.

Then we create the actual funciton that does the work:

```python
import boto3
import re
from pathlib import Path


def lambda_handler(event, context):
    src_key = event["Records"][0]["s3"]["object"]["key"]
    src_bucket = event["Records"][0]["s3"]["bucket"]["name"]
    for pattern, route in S3Router.items():
        if re.compile(pattern).match(src_key):
            dest_bucket = route[0]
            dest_key = src_key if not route[1] else route[1](src_key)

            print(
                "Copying {}/{} to {}/{}".format(
                    src_bucket, src_key, dest_bucket, dest_key
                )
            )
            s3 = boto3.resource("s3")
            copy_source = {"Bucket": src_bucket, "Key": src_key}
            s3.meta.client.copy(copy_source, dest_bucket, dest_key)

```

For any incoming s3 even, this function tries every match in your `S3Router`. If it finds a match, it either resuses the key or uses the function you passed to create the new key. Finally it copies the file to the new desination.

This flow will run in the background without any intervention, though I'd recommend you set up a Cloudwatch log alarm on this function failing.

Now your staging and prod FTP nodes can use the same code, but just have different login information provided.
