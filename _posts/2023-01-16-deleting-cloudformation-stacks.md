---
layout: post
title: "TIL: Deleting a CloudFormation Stack with Missing Roles"
date: 2023-01-16 08:22:00 -0500
categories: til aws
---

Being new to AWS (and, really, all things cloud), I'm prone to blunders. Combined with an unfortunate propensity to delete things I don't understand, I can bork stuff up pretty bad. The latest instance occurred when working through a CDK hello-world demo and I ran into this when running `cdk deploy`:

```shell
Stack Deployments Failed: ValidationError: Role arn:aws:iam::123456789123:role/cdk-hnb659fds-cfn-exec-role-123456789123-us-east-1 is invalid or cannot be assumed
```

Sure enough, looking at IAM on the AWS Console confirmed that the role in question did not exist. Some more looking around revealed I had crashed through several CDK demos, including the one I was working on, six months prior and had deleted most of the roles in the interim. So I decided to delete all CDK-related roles and CloudFormation Stacks for a clean slate. Except...you can't delete a CloudFormation Stack if it does not have a role that allows for deletion. This happens enough that [AWS has an article](https://aws.amazon.com/premiumsupport/knowledge-center/cloudformation-role-arn-error/) on this. It instructs you create a role with the same name as the missing one and make sure it has the required permissions to perform the delete operations on resources in your stack. Great. Except I don't know what those permissions should be. I'm in the "hello-world" phase after all.

My solution was a mashup of the AWS article, a [stack overflow post](https://stackoverflow.com/questions/48709423/unable-to-delete-cfn-stack-role-is-invalid-or-cannot-be-assumed), and trial and error. 

1. I created a new role (named `cloudformation-full-access`) and attached the following policies:
    - `AWSCloudFormationFullAccess`: this made sense
    - `AmazonVPCFullAccess`: this was trial and error and is probably specific to the Stack I was deleting
3. I ran `aws cloudformation delete-stack --role-arn arn:aws:iam::123456789123:role/cloudformation-full-access --stack-name StackToDelete` and confirmed via the AWS Console that the Stack was deleted. Excellent.
4. Re-ran `cdk bootstrap` and, hey, it's actually creating new resources now!
5. Re-ran `cdk deploy`. Success.

I generally keep things like this in my BOLOK (Book Of Lack Of Knowledge), which happens to be a Google Doc right now, but perhaps I'll make posts of them going forward. We'll see.
