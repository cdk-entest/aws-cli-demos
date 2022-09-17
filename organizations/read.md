---
title: AWS Organizations and SCP
author: haimtran
description: simple demo scp
publishedDatte: 17/09/2022
date: 2022-09-17
---

## Introduction

SCP enforce policies into multiple accounts within an OU or multiple OUs. For example, dev department can only launch t2.small instances.

- create multiple accounts
- create organization units
- move accounts into OU
- apply SCP (allow and deny)

## Basic Organizations CLI

list active accounts

```bash
aws organizations list-accounts \
  --query '[Accounts[?Status==`ACTIVE`]]'
```

list root organization

```bash
aws organizations list-roots
```

take note the parent id

```bash
export PARENT_ID=xxxx
```

list organization units

```bash
aws organizations list-organizational-units-for-parent \
  --parent-id $PARENT_ID
```

## Create Accounts and OUs

create an account in each OU

```bash
aws organizations create-account \
      --email xxxx@xxxx.com \
      --account-name scp-demo1
```

take note dev account id

```bash
export DEV_ACCOUNT_ID = xxxx
```

create security and dev OUs and take note OU id

```bash
aws organizations create-organizational-unit \
  --parent-id $PARENT_ID \
  --name Dev
```

```bash
export DEV_OU_ID = xxx
```

move an account into an OU

```bash
aws organizations move-account \
  --account-id $DEV_ACCOUNT_ID \
  --source-parent-id $PARENT_ID \
  --destination-parent-id $DEV_OU_ID
```

close an account

```bash
aws organizations close-account \
  --account-id $DEV_ACCOUNT_ID
```

## Apply SCP to OUs

require dev to use specific ec2 instance type

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireMicroInstanceType",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": ["arn:aws:ec2:*:*:instance/*"],
      "Condition": {
        "StringNotEquals": {
          "ec2:InstanceType": "t2.micro"
        }
      }
    }
  ]
}
```

create a scp

```bash
aws organizations create-policy \
      --content file://policy.json \
      --name SCPPolicyDemoFromCli \
      --type SERVICE_CONTROL_POLICY \
      --description test
```

take note policy id

```bash
export POLICY_ID=xxxx
```

attach scp policy to an OU

```bash
aws organizations attach-policy \
  --policy-id $POLICY_ID \
  --target-id $DEV_OU_ID \

```

deny dev to create dynamodb db table

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyCreateDDBTable",
      "Effect": "Deny",
      "Action": "dynamodb:CreateTable",
      "Resource": ["*"]
    }
  ]
}
```

update policy

```bash
aws organizations update-policy \
  --policy-id $POLICY_ID \
  --content file://update_policy.json
```

## References

1. [best practice scp](https://aws.amazon.com/blogs/industries/best-practices-for-aws-organizations-service-control-policies-in-a-multi-account-environment/)

2. [scp examples](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_examples.html)
