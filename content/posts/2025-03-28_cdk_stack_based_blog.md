---
title: "AWS CDK Stack-based Blog"
date: 2025-03-28T00:00:00-06:00
---

## Background

I've used a bunch of technology to host my personal blog, this particular instance has posts back twelve years. When I worked at Rackspace, I played with using CloudFiles to do static hosting instead of paying for a VPS or cloud server.  This was an adequete setup for a while, writing posts and uploading them.  Next was using Pelican and CloudFiles, still pushing the files up to CloudFiles with Swiftly, a CLI tool for uploading to CloudFiles.  Around 2019, I discovered Hugo, as I wanted a simpler setup and some automation. I used that plus CircleCI to build the content and push it up to cloud storage automatically when I pushed a new post.  At this point I was no longer at Rackspace (2017 layoffs), and was using GCP, as I liked it best and was using it at Unity where I was employed.  

Since then, SSL has been made the standard, and even non-tech savvy people understand what it means to not have HTTPS. None of the bucket-based options will allow a CNAME directly to a bucket and SSL.  I've been letting it sit for a while until I had time to understand the solution, and now I have a new one!

## Solution

Main idea is to use an S3 Bucket to store the static content, a CloudFront Distribution to point to it, an AWS generated certificate for the domain, and a DNS record to point all this to.  

## CDK

AWS's Cloud Development Kit (CDK) lets a developer build out their AWS infrastructure as code. Things similar to this exist, like Terraform and CloudFormation.  CDK uses programming languages such as TypeScript, Go, C#, etc to create a program that will generate the CloudFormation stack, and deploy it to AWS. It will let you even make incremental changes and deploy those changes.

### S3 Bucket

Unfortunately, you still need to create a static site bucket, as the CloudFront S3 Static Site Origin will not route bare subdirectories (think /blog/posts) to the correct base page like index.html.  

A sample bucket creation would look like: 

```typescript
// create bucket for site
const bucket = new s3.Bucket(this, "Bucket", {
    bucketName: "blog.my.site,
    removalPolicy: cdk.RemovalPolicy.DESTROY,
    autoDeleteObjects: true,
    publicReadAccess: true,
    blockPublicAccess: new s3.BlockPublicAccess({
        blockPublicPolicy: false,
        blockPublicAcls: false,
        restrictPublicBuckets: false,
        ignorePublicAcls: false,
    }),
    websiteIndexDocument: "index.html",
    websiteErrorDocument: "404.html",
})
```
This will setup the bucket for static website hosting. One thing I like about CDK is now the `bucket` variable is something you can access and do things with, such as adding permissions directly

```typescript
const originAccessIdentity = new cf.OriginAccessIdentity(this, "OriginAccessIdentity");
bucket.grantRead(originAccessIdentity);
```

Next you want to pull your domain from Route53.  It is possible to use other providers, but this setup is going to utilize it since it can access Route53 to create validation for its certificate as well as create the cname for the site.  *Do not* create the domain in Route53 for your site just yet. 

```typescript
// get zone out of route53
const zone = r53.HostedZone.fromHostedZoneAttributes(this, "Zone", {
    hostedZoneId: "xxxxxxxxxxxxx",
    zoneName: "my.site"
});
```

To issue a certificate with AWS Certificate Manager, it can be done by manually entering the data, but why do that when you can automate it including the DNS validation!

```typescript
// create new certificate for the domain
const certificate = new acm.Certificate(this, "Cert", {
    domainName: "blog.my.site",
    certificateName: "site cert",
    validation: acm.CertificateValidation.fromDns(zone)
});

certificate.applyRemovalPolicy(cdk.RemovalPolicy.DESTROY)
```

With those pieces established, we are going to start to glue together some of this with the CloudFront Distribution

```typescript
// cloudfront distribution for the s3 bucket.
const distribution = new cf.Distribution(this, "Site Distribution", {
    certificate: certificate,
    domainNames: ["blog.my.site"],
    minimumProtocolVersion: cf.SecurityPolicyProtocol.TLS_V1_2_2021,
    defaultBehavior: {
    origin: new cforigin.HttpOrigin(bucket.bucketWebsiteDomainName, {
        protocolPolicy: cf.OriginProtocolPolicy.HTTP_ONLY,
    }),
    }
});
```

This will create a cdn endpoint which will front and cache the files served up when requested.  In addition it will attach the certificate to be served up to the client. 

The last component is to add the record linking our cloudfront distribution to the host or domain we have set aside: 

```typescript
// add a record for t
new r53.ARecord(this, 'SiteAliasRecord', {
    zone: zone,
    recordName: "blog.my.site,
    target: r53.RecordTarget.fromAlias(new r53target.CloudFrontTarget(distribution))
});
```

this blog is now running with this stack! I use github actions to build the hugo site and store the "public" directory as an artifact of the build. That then is used by the deploy job to upload it to an S3 bucket.  Finally, I call the CloudFront create invalidation command to invalidate the cache for the site, so the new content is immediately available. 

```yaml
  deploy_www:
    name: "deploy www site"
    runs-on: ubuntu-latest
    needs: build
    permissions:
      contents: 'read'
      id-token: 'write'
    steps:
      - name: Download public artifacts
        uses: actions/download-artifact@v4
        with:
          name: myblog-public
          path: site
      - name: Setup AWS CLI
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      - name: Sync files to S3 bucket
        run: |
          aws s3 sync site s3://${{secrets.AWS_BUCKET}} --delete
      - name: Invalidaiton of CloudFront Cache
        run : | 
          aws cloudfront create-invalidation --distribution-id ${{secrets.AWS_CF_DISTRIBUTION}} --paths '/*'
```

Full deploy [here](https://github.com/sasimpson/blog/blob/main/.github/workflows/hugo-aws.yaml).
