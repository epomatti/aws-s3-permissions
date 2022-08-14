# AWS S3 Permissions & ACL

This exercise will demonstrate different behaviors for S3 permissions and ACLs.

## Setup

Create the baseline bucket. This bucket will have no public access

```sh
# Setup the common variables
region='sa-east-1'
bucket="permissions-sandbox-$region-epomatti"
file='confidential.csv'

# Create the bucket
aws s3api create-bucket \
  --bucket $bucket \
  --region 'sa-east-1' \
  --create-bucket-configuration "LocationConstraint=$region"

# Set ACL to private
aws s3api put-bucket-acl --bucket $bucket --acl "private"

# Block all public access
aws s3api put-public-access-block \
  --bucket $bucket \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

Now upload the file to the bucket:

```
aws s3api put-object --bucket $bucket --key $file --body "./$file"
```

Make sure to check the bucket ACL at the start:

```sh
aws s3api get-bucket-acl --bucket $bucket
```

## ACL Exercise


Trying to PUT a public ACL to an object will fail because it is blocked by the bucket.

```sh
$ aws s3api put-object-acl --bucket $bucket --key $file --acl public-read

An error occurred (AccessDenied) when calling the PutObjectAcl operation: Access Denied
```

Now enable disable the public ACL block with `BlockPublicAcls=false` and try again:

```sh
aws s3api put-public-access-block \
  --bucket $bucket \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
  
aws s3api put-object-acl --bucket $bucket --key $file --acl public-read
```

This command will succeed! However, it would still not be possible to download the file publicly. For that, configuration `IgnorePublicAcls=false` needs to be set:

```sh
aws s3api put-public-access-block \
  --bucket $bucket \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

Now an interesting use case can be applied.

By blocking NEW all CLs with `BlockPublicAcls=true`, access to the existing ACLs is maintained:

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
$ aws s3api put-bucket-policy --bucket $bucket --policy file://bucket-policy.json

An error occurred (AccessDenied) when calling the PutBucketPolicy operation: Access Denied
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
aws s3api delete-bucket --bucket $bucket
```