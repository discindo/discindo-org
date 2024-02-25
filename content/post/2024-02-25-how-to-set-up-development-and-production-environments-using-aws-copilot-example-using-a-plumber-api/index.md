---
title:
  "How to set up development and production environments using AWS Copilot: Example
  using a `plumber` API."
author: "Teo"
date: "2024-02-25"
slug: how-to-set-up-development-and-production-environments-using-aws-copilot-example-using-a-plumber-api
categories: [R, API, plumber, AWS, AWS Copilot, GitHub Actions]
tags: [AWS ECR, AWS IAM, AWS AppRunner, CD/CI, Automation]
subtitle: ""
summary: "In this post I dive deep into setting up a dev/stage/prod environment setup for a `{plumber}` API on AWS AppRunner"
authors: [teo]
lastmod: "2024-02-25T00:12:16-06:00"
featured: yes
image:
  caption: ""
  focal_point: ""
  preview_only: no
projects: []
---

In this post I am documenting step-by-step the process of deploying
dev/stage/prod environments and instances of a `{plumber}` API on
[AWS AppRunner](https://aws.amazon.com/apprunner/) using
[AWS Copilot](https://aws.amazon.com/containers/copilot/). This is an
expanded follow up to a [previous post](https://discindo.org/post/deploying-plumber-api-to-aws-elastic-container-service/)
on the topic.

# Prerequisites

To manage AWS resources, we need the AWS [`command line interface (cli)`](https://aws.amazon.com/cli/).
To build our infrastructure we'll use AWS Copilot. Follow instructions for
installation [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Copilot.html#copilot-install).
Copilot is a command line interface for containerized applications, so we'll
also need a tool to containerize our `{plumber}` API. Typically `docker` which
can be installed following these [instructions](https://docs.docker.com/get-docker/).

# AWS Access and Services

The process described below, including the permissions policies, makes minimal
assumptions about the permissions a user will have on AWS. It should be enough to
get us started even if we did not have access to any of the needed services. Though,
of course we'll still require an account administrator to grant us the access
by attaching policies to our AWS user or group.

During setup and deployment AWS Copilot requires access to multiple AWS services:

- AWS Identity and access management (IAM)
- AWS Elastic container registry (ECR)
- AWS Cloud formation (CNF)
- AWS Simple storage service (S3)
- AWS Security token service (STS)
- AWS Key management service (KMS)
- AWS Systems manager (SSM)
- AWS Tags manager (TAG)

# AWS Setup

Assume `user` with minimal permissions. Added policies will be documented below.
AWS Copilot uses the `AWS_PROFILE` environmental variable and assumes
the `aws cli` has been [configured](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html).
When properly configured, our development machine will have an `.aws` folder with
`config` and `credentials` files defining the different users, regions,
`aws_secret_access_key` and `aws_secret_key_id`.

```
export $AWS_PROFILE=noob
```

## Docker images

### Base image (Dockerfile_base)

This builds the base image, setting up the R environment and dependencies for
the API. Assuming dependencies will not change often, this image can be pushed
to ECR once and then used to rebuild the API image as the API evolves. If
dependencies change, this image would have to be rebuilt and pushed to ECR.

```
docker build -d Dockerfile_base -t "myapi_base" .
```

The code below tags the image with the name provided by AWS when we create
the registry for the base image. Then, it obtains AWS ECR login credentials
and pushes the local image to AWS ECR. This makes it available for AWS Copilot,
as it is needed when AWS Copilot builds our API service docker image.

```
docker tag myapi_base <aws_account_number>.dkr.ecr.<aws_region>.amazonaws.com/myapi_base
aws ecr get-login-password | \
  docker login -u AWS --password-stdin \
  <aws_account_number>.dkr.ecr.<aws_region>.amazonaws.com/myapi
```

### Service image (Dockerfile)

Make sure its `FROM` instruction is the base registry above

```
FROM <aws_account_number>.dkr.ecr.<aws_region>.amazonaws.com/myapi_base
RUN installr -d remotes
RUN mkdir /build_zone
ADD . /build_zone
WORKDIR /build_zone
RUN R -e 'remotes::install_local(upgrade="never")'
RUN rm -rf /build_zone
EXPOSE 5050
CMD  ["R", "-e", "library(myapi); run_api(port = 5050, host = '0.0.0.0')"]
```

## Initialize AWS resources

Initialize the application with `aws copilot`

```
copilot app init myapi-api
```

Initialize environments:

```
copilot env init --name dev --profile noob
copilot env init --name stage --profile noob
copilot env init --name prod --profile noob
```

At this point the deployment `copilot` directory will look like so:

```
copilot/
├── environments
│   ├── dev
│   │   └── manifest.yml
│   ├── prod
│   │   └── manifest.yml
│   └── stage
│       └── manifest.yml
└── .workspace
```

## Deploy

For each environment, copilot will first deploy the environment using
CloudFromation, and then push build and push the Docker image to
Elastic Container Registry, and finally configure AppRunner to make
the service available.

Dev env

```
copilot init -d ./Dockerfile --app myapi-api -n myapi -t "Request-Driven Web Service" -e dev
```

Stage env

```
copilot init -d ./Dockerfile --app myapi-api -n myapi -t "Request-Driven Web Service" -e stage
```

Prod env

```
copilot init -d ./Dockerfile --app myapi-api -n myapi -t "Request-Driven Web Service" -e prod
```

## Secrets

Create the secret

```
copilot secret init
# follow promts
```

Update the application manifest, should look like this:

```
# You can override any of the values defined above by environment.
environments:
  dev:
    variables:
      LOG_LEVEL: debug # Log level for the "test" environment.
    secrets:
      SECRET: /copilot/myapi-api/dev/secrets/SECRET
  stage:
    secrets:
      SECRET: /copilot/myapi-api/stage/secrets/SECRET
  prod:
    secrets:
      SECRET: /copilot/myapi-api/prod/secrets/SECRET
```

Redeploy the service instance for each env

```
copilot svc deploy --env dev
copilot svc deploy --env stage
copilot svc deploy --env prod
```

## AWS permission policies for used services

### CloudFormation

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "cloudformation:DescribeStackSet",
                "cloudformation:CreateStack",
                "cloudformation:GetTemplate",
                "cloudformation:DescribeStackSetOperation",
                "cloudformation:DeleteStack",
                "cloudformation:UpdateStack",
                "cloudformation:DescribeStackResource",
                "cloudformation:UpdateStackSet",
                "cloudformation:CreateChangeSet",
                "cloudformation:DescribeChangeSet",
                "cloudformation:DeleteStackSet",
                "cloudformation:DescribeStacks",
                "cloudformation:TagResource",
                "cloudformation:GetTemplateSummary",
                "cloudformation:ListStackInstances",
                "cloudformation:CreateStackInstances",
                "cloudformation:ExecuteChangeSet",
                "cloudformation:DescribeStackEvents"
            ],
            "Resource": [
                "arn:aws:cloudformation:*:<aws_account_number>:type/resource/*",
                "arn:aws:cloudformation:*:<aws_account_number>:stackset-target/*",
                "arn:aws:cloudformation:*:<aws_account_number>:stackset/*:*",
                "arn:aws:cloudformation:*:<aws_account_number>:stack/*/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudformation:CreateGeneratedTemplate",
                "cloudformation:ListStacks",
                "cloudformation:UpdateGeneratedTemplate",
                "cloudformation:ListStackSets",
                "cloudformation:DescribeGeneratedTemplate",
                "cloudformation:CreateStackSet",
                "cloudformation:ValidateTemplate"
            ],
            "Resource": "*"
        }
    ]
}
```

### ECR

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchGetImage",
        "ecr:CompleteLayerUpload",
        "ecr:DescribeImages",
        "ecr:TagResource",
        "ecr:DescribeRepositories",
        "ecr:BatchDeleteImage",
        "ecr:UploadLayerPart",
        "ecr:ListImages",
        "ecr:InitiateLayerUpload",
        "ecr:DeleteRepository",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage"
      ],
      "Resource": "arn:aws:ecr:*:<aws_account_number>:repository/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:CreateRepository",
        "ecr:DescribeRegistry",
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    }
  ]
}

```

### IAM

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:UpdateAssumeRolePolicy",
        "iam:ListRoleTags",
        "iam:GetPolicy",
        "iam:TagRole",
        "iam:CreateRole",
        "iam:PutRolePolicy",
        "iam:PassRole",
        "iam:CreateServiceLinkedRole",
        "iam:ListAttachedRolePolicies",
        "iam:UpdateRole",
        "iam:ListPolicyTags",
        "iam:ListRolePolicies",
        "iam:GetRolePolicy"
      ],
      "Resource": [
        "arn:aws:iam::<aws_account_number>:role/*",
        "arn:aws:iam::<aws_account_number>:policy/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:ListPolicies",
        "iam:ListRoles"
      ],
      "Resource": "*"
    }
  ]
}

```

### KMS

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:*:<aws_account_number>:key/*"
    }
  ]
}

```

### S3

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObjectAcl",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::*/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutBucketAcl",
        "s3:CreateBucket",
        "s3:ListBucket",
        "s3:GetBucketAcl",
        "s3:DeleteBucket"
      ],
      "Resource": "arn:aws:s3:::*"
    }
  ]
}

```

### STS

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole",
        "sts:AssumeRoleWithWebIdentity"
      ],
      "Resource": "arn:aws:iam::<aws_account_number>:role/*"
    },
    {
      "Effect": "Allow",
      "Action": "sts:GetCallerIdentity",
      "Resource": "*"
    }
  ]
}
```

### SystemsManager

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:PutParameter",
        "ssm:GetParametersByPath",
        "ssm:GetParameters",
        "ssm:GetParameter",
        "ssm:AddTagsToResource"
      ],
      "Resource": "arn:aws:ssm:<aws_region>:<aws_account_number>:parameter/*"
    },
    {
      "Effect": "Allow",
      "Action": "ssm:DescribeParameters",
      "Resource": "*"
    }
  ]
}
```

### TAG (Tag editor)

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "tag:GetResources",
        "tag:GetTagValues",
        "tag:GetTagKeys"
      ],
      "Resource": "*"
    }
  ]
}
```

## GH Actions

Use this tutorial to create a `IAM` role for `GitHub Actions`:
https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/

### Policy for GH Actions role

The policies for GH Actions has reduced permissions. It is added to the role
created above.

#### Trust Relationship

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<aws_account_number>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/*"
                }
            }
        }
    ]
}
```

#### STS

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sts:AssumeRole",
        "sts:GetCallerIdentity",
        "sts:AssumeRoleWithWebIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

### Deploy

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Resource": "arn:aws:ssm:<aws_region>:<aws_account_number>:parameter/copilot/*",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParametersByPath",
        "ssm:GetParameter"
      ]
    },
    {
      "Resource": "arn:aws:cloudformation:<aws_region>:<aws_account_number>:stackset/myapi-infrastructure:*",
      "Effect": "Allow",
      "Action": [
        "cloudformation:ListStackInstances"
      ]
    },
    {
      "Resource": "arn:aws:cloudformation:<aws_region>:<aws_account_number>:stack/*",
      "Effect": "Allow",
      "Action": [
        "cloudformation:DescribeStacks"
      ]
    },
    {
      "Resource": "*",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ]
    },
    {
      "Resource": [
        "arn:aws:ecr:<aws_region>:<aws_account_number>:repository/myapi/*",
        "arn:aws:ecr:<aws_region>:<aws_account_number>:repository/myapi_base"
      ],
      "Effect": "Allow",
      "Action": [
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ]
    }
  ]
}
```

### Deploy to dev/stage GH Action

Deploy to dev/stage is triggered by push or pull request to the corresponding
branches.

```
# This is a basic workflow to help you get started with Actions
name: Connect to an AWS role from a GitHub repository

# Controls when the action will run. Invokes the workflow on push events but only for the main branch
on:
  push:
    branches: [dev]
  pull_request:
    branches: [dev]

env:
  AWS_REGION: "<aws_region>" #Change to reflect your Region

# Permission can be added at job level or workflow level
permissions:
  id-token: write # This is required for requesting the JWT
  contents: read # This is required for actions/checkout
jobs:
  DeployService:
    runs-on: ubuntu-latest
    steps:
      - name: Git clone the repository
        uses: actions/checkout@v4
      - name: configure aws credentials
        uses: aws-actions/configure-aws-credentials@v1.7.0
        with:
          role-to-assume: arn:aws:iam::<aws_account_number>:role/GitHubAction-AssumeRoleWithAction #change to reflect your IAM role’s ARN
          role-session-name: GitHub_to_AWS_via_FederatedOIDC
          aws-region: ${{ env.AWS_REGION }}
      # Hello from AWS: WhoAmI
      # - name: Sts GetCallerIdentity
      #   run: |
      #     aws sts get-caller-identity
      - name: Install copilot
        run: |
          mkdir -p $GITHUB_WORKSPACE/bin
          # download copilot
          curl -Lo copilot-linux https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux && \
          # make copilot bin executable
          chmod +x copilot-linux && \
          # move to path
          mv copilot-linux $GITHUB_WORKSPACE/bin/copilot && \
          # add to PATH
          echo "$GITHUB_WORKSPACE/bin" >> $GITHUB_PATH
          # - run: copilot help
      - name: deploy service
        run: copilot svc deploy --env dev

```

### Deploy to prod GH Action

Deploy to PROD is triggered by manually creating a release in GitHub.

```
# This is a basic workflow to help you get started with Actions
name: Connect to an AWS role from a GitHub repository

# Controls when the action will run. Invokes the workflow on push events but only for the main branch
on:
  release:
    types: [published]

env:
  AWS_REGION: "<aws_region>" #Change to reflect your Region

# Permission can be added at job level or workflow level
permissions:
  id-token: write # This is required for requesting the JWT
  contents: read # This is required for actions/checkout
jobs:
  DeployService:
    runs-on: ubuntu-latest
    steps:
      - name: Git clone the repository
        uses: actions/checkout@v4
      - name: configure aws credentials
        uses: aws-actions/configure-aws-credentials@v1.7.0
        with:
          role-to-assume: arn:aws:iam::<aws_account_number>:role/GitHubAction-AssumeRoleWithAction #change to reflect your IAM role’s ARN
          role-session-name: GitHub_to_AWS_via_FederatedOIDC
          aws-region: ${{ env.AWS_REGION }}
      # Hello from AWS: WhoAmI
      # - name: Sts GetCallerIdentity
      #   run: |
      #     aws sts get-caller-identity
      - name: Install copilot
        run: |
          mkdir -p $GITHUB_WORKSPACE/bin
          # download copilot
          curl -Lo copilot-linux https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux && \
          # make copilot bin executable
          chmod +x copilot-linux && \
          # move to path
          mv copilot-linux $GITHUB_WORKSPACE/bin/copilot && \
          # add to PATH
          echo "$GITHUB_WORKSPACE/bin" >> $GITHUB_PATH
          # - run: copilot help
      - name: deploy service
        run: copilot svc deploy --env prod

```

# Summary

A step-by-step guide to set up dev/stage/prod environments on AWS for deploying
a `{plumber}` API on AWS AppRunner and setting up GitHub Actions workflows
for automated deployments on the created AWS infrastructure.
