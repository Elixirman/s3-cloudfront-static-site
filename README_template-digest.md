# AWS Static Website Hosting with S3 + CloudFront + OAC

## Project Overview

This project uses **AWS CloudFormation** to automatically deploy a secure static website architecture using:

- **Amazon S3** — stores the website files
- **Amazon CloudFront** — publicly delivers the website through a CDN
- **Origin Access Control (OAC)** — securely allows CloudFront to access the private S3 bucket
- **S3 Bucket Policy** — permits only the specific CloudFront distribution to read website objects
- **HTTPS** — protects communication between visitors and CloudFront
- **CloudFormation** — automates the entire infrastructure deployment

The architecture is:

```text
                    INTERNET
                       │
                       │ HTTPS
                       ▼
              ┌─────────────────┐
              │   CloudFront    │
              │   CDN + HTTPS   │
              └────────┬────────┘
                       │
                  OAC authentication
                  + signed request
                       │
                       ▼
              ┌─────────────────┐
              │   S3 Bucket     │
              │     PRIVATE     │
              │                 │
              │   index.html   │
              │   error.html   │
              │   style.css    │
              │   script.js    │
              └─────────────────┘
```

The key security principle is:

> **Users access the website through CloudFront, not directly through the S3 bucket.**

---

# 1. CloudFormation Template Header

```yaml
AWSTemplateFormatVersion: '2010-09-09'
```

This tells AWS that the file uses the CloudFormation template format.

The date `2010-09-09` is the CloudFormation template format identifier. It does not mean that the resources were created in 2010.

---

# 2. Description

```yaml
Description: >
  Project 1 - Static Website Hosting with S3 + CloudFront.
  Private S3 bucket, publicly served via CloudFront using Origin Access Control (OAC).
```

This is simply a description of the CloudFormation stack.

It tells us that:

1. The website is static.
2. Website files are stored in S3.
3. CloudFront serves the website publicly.
4. The S3 bucket remains private.
5. CloudFront uses OAC to securely access S3.

The `>` is YAML syntax that allows a multi-line description to be treated as one text value.

---

# 3. Parameters

```yaml
Parameters:
  BucketNamePrefix:
    Description: 'Prefix for the S3 bucket (must be globally unique across ALL of AWS)'
    Type: String
    Default: my-portfolio-site
```

Parameters are **inputs or variables** that can be supplied when deploying a CloudFormation template.

Here, the parameter is:

```text
BucketNamePrefix
```

Its type is:

```yaml
Type: String
```

That means the value is text.

The default value is:

```text
my-portfolio-site
```

Therefore, unless another value is supplied, CloudFormation uses:

```text
my-portfolio-site
```

as the bucket name prefix.

## Why is the prefix useful?

S3 bucket names must be **globally unique across AWS**.

For example, this name may already exist:

```text
my-portfolio-site
```

The template therefore combines the prefix with the AWS account ID:

```text
my-portfolio-site-123456789012
```

This greatly reduces the chance of a naming conflict.

---

# 4. Resources

```yaml
Resources:
```

The `Resources` section contains the AWS infrastructure that CloudFormation will create.

This template creates four major resources:

```text
1. S3 Bucket
2. CloudFront Origin Access Control
3. CloudFront Distribution
4. S3 Bucket Policy
```

---

# 5. S3 Bucket

```yaml
SiteBucket:
  Type: 'AWS::S3::Bucket'
```

`SiteBucket` is the logical name CloudFormation uses for the S3 resource.

The resource type:

```yaml
AWS::S3::Bucket
```

tells CloudFormation:

> Create an Amazon S3 bucket.

---

# 6. S3 Bucket Name

```yaml
Properties:
  BucketName: !Sub '${BucketNamePrefix}-${AWS::AccountId}'
```

The `!Sub` function performs string substitution.

The template combines:

```text
BucketNamePrefix
+
AWS Account ID
```

For example:

```text
BucketNamePrefix = my-portfolio-site
AWS Account ID   = 123456789012
```

The resulting bucket name becomes:

```text
my-portfolio-site-123456789012
```

So the final architecture might contain:

```text
S3 Bucket
└── my-portfolio-site-123456789012
```

---

# 7. Making the S3 Bucket Private

```yaml
PublicAccessBlockConfiguration:
  BlockPublicAcls: true
  BlockPublicPolicy: true
  IgnorePublicAcls: true
  RestrictPublicBuckets: true
```

This is one of the most important security sections.

It prevents the S3 bucket from being publicly accessible.

## `BlockPublicAcls`

```yaml
BlockPublicAcls: true
```

Prevents public access through S3 ACLs.

## `BlockPublicPolicy`

```yaml
BlockPublicPolicy: true
```

Prevents policies that attempt to make the bucket publicly accessible.

## `IgnorePublicAcls`

```yaml
IgnorePublicAcls: true
```

Tells S3 to ignore public ACL settings.

## `RestrictPublicBuckets`

```yaml
RestrictPublicBuckets: true
```

Restricts access from policies that would make the bucket public.

### Result

The bucket remains:

```text
             S3
      ┌───────────────┐
      │    PRIVATE    │
      │               │
      │ index.html    │
      │ style.css     │
      │ script.js     │
      │ images/       │
      └───────────────┘
```

The public does not directly access S3.

---

# 8. CloudFront Origin Access Control

```yaml
SiteOAC:
  Type: 'AWS::CloudFront::OriginAccessControl'
```

OAC stands for:

> **Origin Access Control**

It allows CloudFront to securely access a private S3 origin.

Think of OAC as an authentication mechanism between CloudFront and S3.

Without OAC, CloudFront would not be able to access the private bucket.

The relationship becomes:

```text
CloudFront
     │
     │ OAC
     ▼
Private S3
```

---

# 9. OAC Configuration

```yaml
OriginAccessControlConfig:
```

This section defines how the OAC operates.

## OAC Name

```yaml
Name: !Sub '${BucketNamePrefix}-oac'
```

If the bucket prefix is:

```text
my-portfolio-site
```

the OAC might be called:

```text
my-portfolio-site-oac
```

This is mainly a descriptive name.

---

# 10. Origin Type

```yaml
OriginAccessControlOriginType: s3
```

This tells CloudFront:

> The origin being protected is an S3 bucket.

Remember:

**Origin = the place where CloudFront gets the actual content.**

In this project:

```text
CloudFront
     │
     ▼
    S3
```

S3 is the origin.

---

# 11. Signing Behavior

```yaml
SigningBehavior: always
```

This tells CloudFront:

> Always sign requests sent to the S3 origin.

Conceptually:

```text
CloudFront
    │
    │ "Give me index.html"
    │
    │ + AWS authentication/signature
    ▼
   S3
```

S3 can verify that the request is authorized.

---

# 12. Signing Protocol

```yaml
SigningProtocol: sigv4
```

This specifies **AWS Signature Version 4 (SigV4)**.

CloudFront uses SigV4 to authenticate requests to S3.

You do not have to manually create these signatures.

AWS handles the signing process.

---

# 13. CloudFront Distribution

```yaml
SiteDistribution:
  Type: 'AWS::CloudFront::Distribution'
```

This creates the CloudFront distribution.

CloudFront acts as the public front door to the website.

Instead of:

```text
User → S3
```

the architecture is:

```text
User → CloudFront → S3
```

This provides:

- CDN delivery
- HTTPS
- caching
- better global performance
- secure access to the private S3 bucket

---

# 14. Enable CloudFront

```yaml
Enabled: true
```

This activates the CloudFront distribution.

---

# 15. Default Root Object

```yaml
DefaultRootObject: index.html
```

This tells CloudFront what file to serve when the user visits the root of the website.

For example:

```text
https://example.com/
```

CloudFront looks for:

```text
index.html
```

The relationship is:

```text
/
↓
index.html
```

---

# 16. HTTP Version

```yaml
HttpVersion: http2
```

This enables HTTP/2 support for the CloudFront distribution.

HTTP/2 provides more efficient communication between browsers and CloudFront.

---

# 17. CloudFront Price Class

```yaml
PriceClass: PriceClass_100
```

CloudFront has edge locations distributed around the world.

Price classes determine which geographic edge locations CloudFront uses.

Generally:

```text
More edge locations
        ↓
More geographic coverage
        ↓
Potentially higher cost
```

`PriceClass_100` limits the distribution to a subset of locations and is useful when trying to control costs.

For a portfolio project, this can be a reasonable cost-conscious choice.

---

# 18. Default Cache Behavior

```yaml
DefaultCacheBehavior:
```

This section defines how CloudFront handles requests from website visitors.

---

## Target Origin

```yaml
TargetOriginId: SiteS3Origin
```

This tells CloudFront:

> Send requests to the origin named `SiteS3Origin`.

The origin itself is defined later in the template.

---

# 19. Redirect HTTP to HTTPS

```yaml
ViewerProtocolPolicy: redirect-to-https
```

This forces visitors toward HTTPS.

For example:

```text
http://example.com
```

is redirected to:

```text
https://example.com
```

The flow is:

```text
HTTP
 ↓
CloudFront
 ↓
HTTPS
```

This helps protect data transmitted between the visitor and CloudFront.

---

# 20. Allowed Methods

```yaml
AllowedMethods: [GET, HEAD]
```

A static website normally only needs users to retrieve files.

`GET` means:

> Give me this file.

Examples:

```text
GET /index.html
GET /style.css
GET /script.js
GET /images/logo.png
```

`HEAD` means:

> Give me information about the object without returning its content.

The website does not need methods such as:

```text
POST
PUT
DELETE
PATCH
```

because it is a static website.

---

# 21. Cached Methods

```yaml
CachedMethods: [GET, HEAD]
```

CloudFront caches GET and HEAD requests.

This is what makes CloudFront function as a CDN.

For example:

### First visitor

```text
User 1
  ↓
CloudFront
  ↓
S3
  ↓
index.html
```

CloudFront can cache the file.

### Second visitor

```text
User 2
  ↓
CloudFront
  ↓
Cached index.html
```

CloudFront may serve the cached file without requesting it from S3 again.

This can improve performance and reduce requests to S3.

---

# 22. Cache Policy

```yaml
CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
```

This specifies a CloudFront cache policy.

A cache policy determines how CloudFront handles caching.

It can influence things such as:

- what is cached
- how requests are cached
- how long content remains cached
- how CloudFront handles request information

The ID refers to an AWS-managed CloudFront cache policy.

---

# 23. CloudFront Origin

```yaml
Origins:
  - Id: SiteS3Origin
```

This defines the origin used by CloudFront.

The logical ID is:

```text
SiteS3Origin
```

Think of this as a label:

```text
SiteS3Origin = S3 website bucket
```

So:

```text
CloudFront
     │
     ▼
SiteS3Origin
     │
     ▼
S3 Bucket
```

---

# 24. S3 Regional Domain Name

```yaml
DomainName: !GetAtt SiteBucket.RegionalDomainName
```

`!GetAtt` means:

> Get an attribute from another CloudFormation resource.

Here:

```yaml
!GetAtt SiteBucket.RegionalDomainName
```

means:

> Get the regional domain name of the S3 bucket called `SiteBucket`.

CloudFormation automatically connects the CloudFront distribution to the S3 bucket.

You do not have to manually type the S3 endpoint.

---

# 25. Attach the OAC

```yaml
OriginAccessControlId: !GetAtt SiteOAC.Id
```

This connects the OAC to CloudFront.

It means:

```text
CloudFront
     │
     │ OAC
     ▼
Private S3
```

CloudFront can now authenticate when requesting objects from S3.

---

# 26. Origin Access Identity

```yaml
S3OriginConfig:
  OriginAccessIdentity: ''
```

This may look strange.

There are two related AWS mechanisms:

- OAI — Origin Access Identity
- OAC — Origin Access Control

OAI is the older mechanism.

This template uses the newer **OAC**.

Therefore:

```yaml
OriginAccessIdentity: ''
```

is left empty because the template is using OAC instead.

---

# 27. Custom Error Response

```yaml
CustomErrorResponses:
  - ErrorCode: 404
    ResponseCode: 200
    ResponsePagePath: /error.html
```

This controls what happens when CloudFront receives a 404 error.

Suppose someone visits:

```text
https://example.com/does-not-exist
```

Normally:

```text
404 Not Found
```

Instead, CloudFront serves:

```text
/error.html
```

The template also tells CloudFront to return:

```text
200 OK
```

instead of:

```text
404 Not Found
```

### Important consideration

Returning `200` for an error page is not always desirable.

If `error.html` is simply a normal 404 page, returning a real `404` status is usually more semantically correct.

Returning `200` can be intentional for applications such as single-page applications where client-side routing needs a fallback page.

---

# 28. S3 Bucket Policy

```yaml
SiteBucketPolicy:
  Type: 'AWS::S3::BucketPolicy'
```

This creates an IAM-style policy attached to the S3 bucket.

The policy answers:

> Who is allowed to access the bucket, and what can they do?

---

# 29. Which Bucket Does the Policy Apply To?

```yaml
Bucket: !Ref SiteBucket
```

`!Ref SiteBucket` means:

> Use the S3 bucket created by the `SiteBucket` resource.

CloudFormation automatically connects the policy to the bucket.

---

# 30. Policy Version

```yaml
PolicyDocument:
  Version: '2012-10-17'
```

This specifies the AWS IAM policy language version.

It is standard syntax used in AWS policies.

---

# 31. Policy Statement

```yaml
Statement:
  - Sid: AllowCloudFrontServicePrincipalReadOnly
```

A statement is one permission rule.

The `Sid` is simply a descriptive identifier.

The name:

```text
AllowCloudFrontServicePrincipalReadOnly
```

means:

> Allow the CloudFront service to read objects.

---

# 32. Effect

```yaml
Effect: Allow
```

This grants permission.

The rule says:

```text
ALLOW
```

rather than:

```text
DENY
```

---

# 33. Principal

```yaml
Principal:
  Service: cloudfront.amazonaws.com
```

This specifies **who** receives the permission.

The principal is:

```text
cloudfront.amazonaws.com
```

Therefore:

> The CloudFront service is allowed to access the S3 bucket.

It does NOT say:

```yaml
Principal: "*"
```

which would represent everyone.

The security model is therefore:

```text
Internet
   │
   ✖
   ▼
 S3

CloudFront
   │
   ✓
   ▼
 S3
```

---

# 34. Action

```yaml
Action: 's3:GetObject'
```

This specifies what CloudFront is allowed to do.

CloudFront can:

> Read/download objects from S3.

For example:

```text
index.html
style.css
script.js
logo.png
```

It does not receive permission to:

```text
Delete objects
Upload objects
Modify objects
Create buckets
```

This follows the **principle of least privilege**.

---

# 35. Resource

```yaml
Resource: !Sub '${SiteBucket.Arn}/*'
```

This specifies which resources CloudFront can access.

`${SiteBucket.Arn}` gets the bucket's Amazon Resource Name.

The `/*` means:

> All objects inside the bucket.

For example:

```text
S3 Bucket
│
├── index.html
├── error.html
├── style.css
├── script.js
└── images/
    └── logo.png
```

CloudFront can read the objects within the bucket.

---

# 36. Source ARN Condition

```yaml
Condition:
  StringEquals:
    'AWS:SourceArn': !Sub 'arn:aws:cloudfront::${AWS::AccountId}:distribution/${SiteDistribution}'
```

This is one of the strongest security controls in the template.

It says:

> Only allow this specific CloudFront distribution to access the S3 bucket.

It does not simply trust every CloudFront distribution.

It identifies the exact distribution using its ARN.

Conceptually:

```text
CloudFront Distribution A
        │
        ✖
        │
        ▼
       S3


CloudFront Distribution B
        │
        ✖
        │
        ▼
       S3


Your CloudFront Distribution
        │
        ✓
        │
        ▼
       S3
```

This provides a tighter security boundary.

---

# 37. Outputs

```yaml
Outputs:
```

Outputs are values CloudFormation displays after the stack has been successfully deployed.

They make important information easy to retrieve.

This template outputs:

1. S3 bucket name
2. CloudFront URL
3. CloudFront distribution ID

---

# 38. Bucket Name Output

```yaml
BucketName:
  Description: 'Name of the S3 bucket holding site files'
  Value: !Ref SiteBucket
```

After deployment, CloudFormation might show:

```text
BucketName
my-portfolio-site-123456789012
```

This tells you where the website files are stored.

---

# 39. CloudFront URL

The original template contains a formatting problem here:

```yaml
Value: !Sub '[https://${SiteDistribution.DomainName](https://${SiteDistribution.DomainName)}'
```

That should be:

```yaml
Value: !Sub 'https://${SiteDistribution.DomainName}'
```

CloudFormation will then output something similar to:

```text
https://d1234567890abcd.cloudfront.net
```

This is the public CloudFront URL for the website.

---

# 40. Distribution ID

```yaml
DistributionId:
  Description: 'CloudFront distribution ID'
  Value: !Ref SiteDistribution
```

This outputs the CloudFront distribution ID.

For example:

```text
E123ABC456XYZ
```

The distribution ID is useful when managing or updating the CloudFront distribution.

---

# 41. The Complete Architecture

The entire project can be summarized as:

```text
                         INTERNET
                            │
                            │ HTTPS
                            ▼
                  ┌─────────────────────┐
                  │      CLOUDFRONT     │
                  │                     │
                  │  CDN + HTTPS        │
                  │  Caching            │
                  │  OAC Authentication │
                  └──────────┬──────────┘
                             │
                             │ Signed AWS Request
                             │
                             ▼
                  ┌─────────────────────┐
                  │         S3          │
                  │                     │
                  │       PRIVATE       │
                  │                     │
                  │   index.html        │
                  │   error.html        │
                  │   style.css         │
                  │   script.js         │
                  │   images/           │
                  └─────────────────────┘
```

---

# 42. What Happens When a User Visits the Website?

Suppose the user visits:

```text
https://d1234567890abcd.cloudfront.net
```

### Step 1 — User connects to CloudFront

```text
User
  │
  │ HTTPS
  ▼
CloudFront
```

CloudFront receives the request.

---

### Step 2 — CloudFront checks its cache

CloudFront checks whether it already has the requested file cached.

If it does:

```text
CloudFront Cache
      │
      ▼
    User
```

The request can be served immediately.

---

### Step 3 — If the file is not cached

CloudFront contacts S3.

```text
CloudFront
     │
     │ Signed request
     │ OAC
     ▼
Private S3
```

---

### Step 4 — S3 verifies the request

S3 checks the bucket policy.

The policy asks:

1. Is the request from CloudFront?
2. Is it from the authorized CloudFront distribution?
3. Is CloudFront requesting only `s3:GetObject`?

If everything is allowed:

```text
S3
 │
 │ index.html
 ▼
CloudFront
```

---

### Step 5 — CloudFront sends the file to the user

```text
S3
 │
 ▼
CloudFront
 │
 │ HTTPS
 ▼
User
```

CloudFront can also cache the object for future visitors.

---

# 43. Why Use This Architecture?

## Security

The S3 bucket is private.

Users do not need direct S3 access.

Only the authorized CloudFront distribution can read objects.

---

## Performance

CloudFront caches website files at edge locations.

Users can receive content from a location geographically closer to them.

---

## HTTPS

CloudFront can serve the website through HTTPS.

HTTP requests can be redirected to HTTPS.

---

## Scalability

S3 and CloudFront are managed AWS services.

You don't have to manage:

- physical servers
- operating systems
- web servers
- load balancers
- disk capacity

for this basic static website.

---

## Automation

CloudFormation lets you define the infrastructure as code.

Instead of manually creating everything in the AWS Console, you deploy the YAML template.

---

# 44. Important CloudFormation Functions Used

This template uses several CloudFormation intrinsic functions.

## `!Sub`

Example:

```yaml
!Sub '${BucketNamePrefix}-${AWS::AccountId}'
```

Meaning:

> Substitute variables into a string.

---

## `!Ref`

Example:

```yaml
!Ref SiteBucket
```

Meaning:

> Reference another CloudFormation resource or parameter.

---

## `!GetAtt`

Example:

```yaml
!GetAtt SiteBucket.RegionalDomainName
```

Meaning:

> Get a specific attribute from another resource.

---

# 45. Simple Meaning of the Main Components

| Component | Simple Meaning |
|---|---|
| CloudFormation | Builds the infrastructure automatically |
| S3 | Stores website files |
| S3 Private Bucket | Keeps website files protected from direct public access |
| CloudFront | Delivers the website to users |
| CDN | Caches content closer to users |
| OAC | Allows CloudFront to securely authenticate to S3 |
| Bucket Policy | Defines who can access S3 |
| `GetObject` | Read an object from S3 |
| HTTPS | Encrypts traffic between the user and CloudFront |
| Cache Policy | Controls CloudFront caching |
| Origin | Where CloudFront gets the content |
| Output | Shows useful information after deployment |

---

# 46. The Most Important Concept to Remember

For an interview or cloud architecture explanation, remember this:

```text
S3 = STORAGE
CloudFront = DELIVERY
OAC = SECURE CONNECTION
Bucket Policy = PERMISSION
HTTPS = ENCRYPTION
CloudFormation = AUTOMATION
```

Or even shorter:

> **CloudFormation creates it → S3 stores it → OAC protects access → CloudFront delivers it → HTTPS secures the visitor connection.**

---

# 47. Final Security Model

The most important security flow is:

```text
                    PUBLIC INTERNET
                           │
                           ▼
                  ┌─────────────────┐
                  │   CloudFront    │
                  │                 │
                  │ HTTPS           │
                  │ CDN             │
                  │ OAC             │
                  └────────┬────────┘
                           │
                           │ Authorized
                           │ Signed Request
                           ▼
                  ┌─────────────────┐
                  │       S3        │
                  │                 │
                  │    PRIVATE      │
                  │                 │
                  │  Website Files  │
                  └─────────────────┘
```

Therefore:

```text
Internet ──X──> S3
Internet ──────> CloudFront ──✓──> S3
```

This is the central security principle of the project.
