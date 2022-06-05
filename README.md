# Cross-region pipeline for AWS CloudFront + WAF v2 (with OWASP Top 10 rules)

*Disclaimer 1: This solution intends to provide value for everyone from beginners to experienced AWS users. However, the documentation provided does assume a fair bit of knowledge of the AWS world. In any case, please use the materials as a sample / starting point only and NOT as a production-ready system, as important things such as least-privilege permissions are not enforced.*

*Disclaimer 2: The deployed resources per se will not cost you any money, as they are either free or fall under the free tier. However they **may start having a cost involved** if you start utilising them, for instance using the CloudFront distribution to retrieve files from the origin S3 bucket. Do so at your own peril.*

## Solution overview

*For a full walkthrough, including a solution diagram, please read [my Medium blog post](https://medium.com/@misterjunio/aws-cross-region-codepipeline-for-cloudfront-waf-v2-2f9f943cc935).*

This project presents a solution for the following problem: you have a pipeline defined in CodePipeline on a given region other than `us-east-1` which deploys, among other resources, a CloudFront distribution. You want to attach to the distribution a Web ACL from the WAF service that is now in v2. The issue is, the WAF v2 only allows you to deploy global ACLs (i.e. targeted at CloudFront) in the `us-east-1` region.

As such, because your deployment pipeline and the ACL have to live in separate regions, you cannot use the typical CloudFormation "Export-then-ImportValue" pattern; that only works for stacks in the same region. You *could* create the WAF completely separately and then feed its ARN as an input to the stack that deploys CloudFront, but that would involve manual work, which you don't like because it goes against your objective of full automation via pipelines.

Hence the goal of my solution. Its main components are 3 CloudFormation stacks:

- CI/CD pipeline stack, backed by the `cicd.yaml` template. It defines the pipeline responsible for creating/updating the CI/CD pipeline itself, the WAF stack and the main (CloudFront) stack.

- WAF stack, backed by the `waf-v2-owasp-top-10.yaml` template. This is an added bonus, my personal adaptation of [this archive](https://github.com/amazon-archives/aws-waf-sample/tree/master/waf-owasp-top-10) for the most recent and recommended v2 of AWS's WAF. Unlike the original one, I simplified it to support only the global `CLOUDFRONT` scope. You could probably easily adapt it to suit a regional load balancer. In that case this stack wouldn't make much sense in the context of this solution though, because such regional Web ACLs can be deployed in any region.

- Main stack, backed by the `main.yaml` template. Once deployed, it creates a Lambda-backed custom CloudFormation resource to fetch the ARN of the Web ACL, a CloudFront distribution to which the ACL is attached, and a "dummy" S3 bucket to act as the distribution origin.

## Setup instructions

These instructions assume you're deploying your CI/CD in a region other than `us-east-1`. As explained in the previous sections, that's the whole goal.

Before you start, it's recommended that you configure the AWS CLI. You can do so by following [this documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html). You should also have access to the AWS Console on the same AWS account you're set up for. Ensure you have administrator-level IAM permissions to avoid unexpected access issues. Tightening it up is one of the things that can use more attention later if you're thinking of going to production.

1. Clone this repository.

1. Create an S3 bucket **in the `us-east-1` region**, to hold the artifacts from CodePipeline when deploying the WAF to this region. You can name it anything you like, but make sure you write down the name. Do it either via Console (all the default options are OK) or use the sample command

    ``` sh
    aws s3api create-bucket --bucket [bucket-name] --region us-east-1
    ```

1. Find the `parameters/cicd.json` file and add the name of the S3 bucket you entered on the previous step as the value for the `pWafStackArtifactsS3BucketName` key instead of the default one.

1. Replace all other parameter values on the JSON files inside the `parameters` folder (`cicd.json`, `waf-v2-owasp-top-10.json` and `main.json`), optionally adding tags. You may also want to add stack policies later on. The parameter names should be quite self-explanatory but if you need more information, refer to the descriptions in the `YAML` templates.

1. Create the CI/CD stack.

    - If you have configured the AWS CLI, run the following command to deploy the solution, replacing `misterjunio-waf-v2-cicd-stack` with your preferred stack name:

    ``` sh
    aws cloudformation deploy --template-file cicd.yaml --stack-name misterjunio-waf-v2-cicd-stack --parameter-overrides file://parameters/cicd.json --capabilities CAPABILITY_NAMED_IAM
    ```

    - If you haven't, then alternatively use the AWS Console to navigate to CloudFormation and create a new stack from the `cicd.yaml` template. In this case, ensure that the parameter values you set match the ones you filled in the `parameters/cicd.json` file.

1. Wait for the stack to deploy, which should take only a couple of minutes. Hope everything goes smoothly :crossed_fingers:

1. Follow [this guide](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-gc.html) on setting up CodeCommit credentials for the IAM user created by the stack. The user name is per what you defined for `pCodeCommitUserName` in the parameters.

1. Most likely you'll be using Git via HTTPS to interact with CodeCommit. If so, run the following commands to set your new repo as the remote and push the solution into its `main` branch, replacing `[your-repo-name]` with the `pCodeCommitRepoName` parameter value you chose:

    ``` sh
    git remote set-url origin https://git-codecommit.ap-southeast-2.amazonaws.com/v1/repos/[your-repo-name]
    git push -u origin main
    ```

1. Navigate to CodePipeline and open the pipeline with name as per your `pCodePipelineName` parameter value. At this point it should shortly recognise the push to CodeCommit and automatically run the stack deployments.

1. Wait for the entire pipeline to succeed. You should finally have everything spun up. You can check out the juicy stuff via the Console:

    - A CloudFront distribution in whatever region you deployed the CI/CD stack (with an origin S3 bucket in that same region).
    - A WAF v2 Web ACL in the `Global (CloudFront)` region.
    - An association between the Web ACL and the CloudFront distribution.

Play around with the project at your own pace, tweak whatever you need and enjoy!

## Cleaning up

To remove the entire solution from your AWS account the most important thing is to delete the three stacks that it creates, either via Console or CLI. Note that the order matters: you should delete the stack backed by the `main.yaml` template first, then the WAF one (`waf-v2-owasp-top-10.yaml` template backed, which you'll find in the N. Virginia region) and finally the `cicd.yaml` template one.

If using the CLI, the command to do so is `aws cloudformation delete-stack --stack-name [stack-name]`, where `[stack-name]` is the name of each stack. Be careful, because the *stack* names were defined by yourself and most likely differ from the *template* names. Also, don't forget to add `--region us-east-1` when deleting the WAF stack.

A few resources have a "retain" deletion policy to avoid bad states during stack deletion. If you want to clean them up, you'll have to manually remove them, which namely are the IAM user for CodeCommit and a couple of IAM roles as well. You can find the role names in the CI/CD stack outputs, or if you deleted it already then you can sort roles by the latest creation date/time in IAM. Otherwise, the names should be pretty obvious, as they partly include the stack names.

Lastly, head over to S3. CodePipeline artifact-storing buckets are not deleted by default, as they may not (and probably are not) empty. Same as with IAM roles, you can sort by creation date/time plus their names should be clear enough. Empty the buckets before deleting them. You can also opt to empty and delete the S3 bucket you created manually to hold the `us-east-1` CodePipeline artifacts.
