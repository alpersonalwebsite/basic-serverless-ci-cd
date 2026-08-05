# Basic serverless CI/CD

CI/CD for a [Serverless Framework](https://www.serverless.com/) project on CircleCI,
with an S3 static site, a Route 53 alias record, and the certificate ARN held in SSM
Parameter Store.

## Deploying from CI

`serverless.yml` deliberately does **not** set a `profile:`. A named profile only
exists in a local `~/.aws/credentials`, so pinning one made the config unreadable
anywhere else, CI included: the framework fails with `Profile ... does not exist`
before it even prints the config.

Set these in the CircleCI project (or a context) instead:

| Variable | Purpose |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | deploy credentials |
| `AWS_SECRET_ACCESS_KEY` | deploy credentials |
| `AWS_REGION` | `us-east-1` here, matching `provider.region` |
| `CERTIFICATE_ARN` | *optional.* Skips the SSM lookup, so the deploy role does not need `ssm:GetParameter` |

The deploy role needs permission to create the stack's resources, plus
`ssm:GetParameter` on `/AllEnvironments/certificateARN` unless you set
`CERTIFICATE_ARN`. Grant it what the stack actually creates rather than reusing a
broad administrator key.

Locally, pick the profile per command instead of hardcoding it:

```shell
cd src
AWS_PROFILE=serverless-admin npx serverless deploy -v
```

Node is pinned to 12.16 in `.nvmrc` and in the CircleCI executor, matching the
`nodejs12.x` Lambda runtime.

## Bucket

1. Create bucket with your domain name: `anhydride.info`
1. Go to the bucket's properties and enable `Static website hosting`. Set as index, `index.html`
1. Create `index.html` and upload it to the bucket.
1. Go to the bucket's permissions, select `Block public access` and click Edit. Uncheck
   only the two **public bucket policy** settings ("Block public access to buckets and
   objects granted through *new* public bucket policies" and "...through *any* public
   bucket policies"). Leave the two **ACL** settings switched on: the policy below is
   what serves the site, and the ACL route is not needed.
1. Add the policy at the bucket level. Go to `Bucket Policy`, paste and save. This is
   what makes the objects readable, so there is no need to mark files public one by
   one with `Make it public`.
```json
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Sid":"PublicReadGetObject",
         "Effect":"Allow",
         "Principal":"*",
         "Action":[
            "s3:GetObject"
         ],
         "Resource":[
            "arn:aws:s3:::anhydride.info/*"
         ]
      }
   ]
}
```
1. Add an `Alias record` for the domain.
  1. Go to `Route 53`
  1. Choose Hosted Zones.
  1. Click the hosted zone that matches your domain
  1. Click on Create Record Set
  In our case...
  Name:
  Type: A - IPv4 address
  Alias: Yes
  Alias target: choose your bucket

1. You should now be able to reach the site through the static website endpoint.
   Example: `http://anhydride.info.s3-website-us-east-1.amazonaws.com/`

If you go to https://anhydride.info/ you should see your `static website`: Nothing here!

**What this bucket is.** Everything under it is world-readable, which is the point of
a public static site, but be deliberate about it: put only site content in this
bucket. The hardened alternative, if you want the bucket itself private, is to serve
it through CloudFront with an Origin Access Identity and leave all four Block Public
Access settings on. That is a different setup than the one below and is not what this
repo builds.

## Variable in Parameter Store

1. Go to AWS Systems Manager
1. Choose Parameter Store
1. Click on Create parameter
    1. Name: `/AllEnvironments/certificateARN`
    1. Description: Certificate ARN for serverless-domain-manager
    1. Tier: Standard
    1. Type: String
    1. Value: your certificate ARN. Example: `arn:aws:acm:us-east-1:your-aws-account-id:certificate/******`

`serverless.yml` already reads it, so there is nothing to edit:

```yaml
certificateArn: ${env:CERTIFICATE_ARN, ssm:/AllEnvironments/certificateARN}
```

`CERTIFICATE_ARN` takes precedence if it is set, which is how a deploy role without
`ssm:GetParameter` can still work. Note that serverless 1.x resolves both sides of a
fallback concurrently, so it probes SSM either way: with credentials configured the
probe fails fast and the environment value wins, and with no credentials at all you
will see "taking longer than expected" for a while before it settles.