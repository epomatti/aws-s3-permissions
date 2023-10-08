# AWS S3 Permissions & ACL

This exercise will demonstrate different behaviors for S3 permissions, which include:

- Bucket policies
- Bucket ACLs
- IAM Policies

When `BucketOwnerEnforced` is set (which is the default), ACLs are disabled and permissions must be set with policies.

## Setup

Create the sample bucket:

```sh
# Setup the common variables
region='us-east-2'
bucket="bucket-pomatti-permissions"
file='confidential.csv'

# Create the bucket
aws s3api create-bucket --bucket $bucket --region $region \
  --create-bucket-configuration "LocationConstraint=$region"
```

For new S3 buckets, the feature `Block all public access` is enabled by default.

Additionally, as per [ACL overview][1]:

> By default, Object Ownership is set to the bucket owner enforced setting, and all ACLs are disabled. When ACLs are disabled, the bucket owner owns all the objects in the bucket and manages access to them exclusively by using access-management policies.


Now upload the file to the bucket:

```sh
aws s3api put-object --bucket $bucket --key $file --body $file
```

Make sure to check the bucket ACL at the start:

```sh
aws s3api get-bucket-acl --bucket $bucket
```

## ACL Exercise

Trying to PUT a public ACL to an object will fail because ACLs are disabled.

```sh
aws s3api put-object-acl --bucket $bucket --key $file --acl public-read
```

First, enable the bucket ACLs by setting the ownership to `BucketOwnerPreferred`:

```sh
aws s3api put-bucket-ownership-controls \
    --bucket $bucket \
    --ownership-controls="Rules=[{ObjectOwnership=BucketOwnerPreferred}]"
```

Now, disable the public ACL block by setting `BlockPublicAcls=false` and try again:

```sh
aws s3api put-public-access-block \
  --bucket $bucket \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# This should work now
aws s3api put-object-acl --bucket $bucket --key $file --acl public-read
```

This command will succeed. However, it would still not be possible to download the file publicly (anonymously). For that, configuration `IgnorePublicAcls=false` needs to be set:

```sh
aws s3api put-public-access-block \
  --bucket $bucket \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

Now an interesting use case can be applied.

By blocking NEW ACLs with `BlockPublicAcls=true`, access to the existing ACLs is maintained:

```sh
aws s3api put-public-access-block \
  --bucket $bucket \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=false,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

To block everything again, both new and existing ACLs need to be blocked/ignored with `BlockPublicAcls=true,IgnorePublicAcls=true`:

```sh
aws s3api put-public-access-block \
  --bucket $bucket \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

Before moving on let's remove the object public ACL:

```sh
aws s3api put-object-acl --bucket $bucket --key $file --acl private
```

## Bucket Policies

Adding a bucket policy at this stage will fail:

```sh
aws s3api put-bucket-policy --bucket $bucket --policy file://bucket-policy.json
```

To allow it, set `BlockPublicPolicy=false`:

```sh
aws s3api put-public-access-block \
  --bucket $bucket \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=true"
```

Now creating a new bucket policy should work:

```sh
aws s3api put-bucket-policy --bucket $bucket --policy file://bucket-policy.json
```

## Clean-up

```sh
aws s3 rb s3://$bucket --force
```

[1]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/acl-overview.html
