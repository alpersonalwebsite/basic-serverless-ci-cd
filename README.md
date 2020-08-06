## Bucket

1. Create bucket with your domain name: `anhydride.info`
1. Go to the bucket's properties and enable `Static website hosting`. Set as index, `index.html`
1. Create `index.html` and upload it to the bucket.
1. Go to the bucket's permissions, select `Block public access`, click on Edit, **uncheck** all boxes and save.
1. Select `index.html`, click on Actions and `Make it public`
1. You should be able to access to the website through the static site endpoint. Example: `http://anhydride.info.s3-website-us-east-1.amazonaws.com/`
1. If you want to make ALL the files public, you can add a policy at the bucket level. Go to `Bucket Policy`, paste and save.
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

If you go to https://anhydride.info/ you should see your `static website`: Nothing here!

Variable in Parameter Store
1. Go to AWS Systems Manager 
1. Choose Parameter Store
1. Click on Create parameter
  1. Name: /AllEnvironments/certificateARN
  1. Description: Certificate ARN for serverless-domain-manager
  1. Tier: Standard
  1. Type: String
  1. Value: your certificate ARN. Example: arn:aws:acm:us-east-1:your-aws-account-id:certificate/******
1. In `serverless.yml` replace your certificate ARN with `${ssm:/AllEnvironments/certificateARN}`