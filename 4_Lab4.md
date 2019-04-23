
## Lab 4 - Infrastructure as a Code and DevSecOps:

### Stage 1: IaaC
We want to control our infrastrucure in order to enforce security policies. 

1. Let’s make the resources in the Lab become code!

```console
~/environment/WebAppRepo (master) $ cp aws-devops-essential/templates/02-aws-devops-workshop-environment-setup.template ./WebAppRepo/
```

2. Take a few minutes to review in the Cloud9 IDE the template we just copied. Check out some CloudFormation intrinsic functions and features (Mappings, Fn::Join, Ref, etc).

3. Let’s push this template to our CodeCommit Repo:

```console
~/environment/WebAppRepo (master) $ git add 02-aws-devops-workshop-environment-setup.template
~/environment/WebAppRepo (master) $ git commit -m "Added cf"
~/environment/WebAppRepo (master) $ git push origin master
```

4. Create the following stage in our CodePipeline to deploy the infrastructure (VPC, EC2, Security Groups, etc.). Click on "Edit" button on the top right in the CodePipeline console:

5. _Add a new stage to Pipeline after Source and before Build:__

- __Stage Name__: `CreateEnv`

- __Action provider__: `AWS CloudFormation`

- __Action name__: `CreateResources`

- __Input Artifact__: `SourceArtifact`

- __Action mode__: `Create or update a stack`

- __Stack name (select the stack that was created before)__: `DevopsWorkshop-Env`

- __Template__: ```SourceArtifact::02-aws-devops-workshop-environment-setup.template```

- __Capabilities__: `CAPABILITY_IAM`

- __Role name__: select the one that contains `CFNRole` that was created before

- __Output__: `CfOutput`

The configurations should be:

![1-5a](./img/Lab4-Stage-1-5a.png)

![1-5b](./img/Lab4-Stage-1-5b.png)

![1-5c](./img/Lab4-Stage-1-5c.png)

6. On the CodePipeline pipeline click on the “Release” button to check if the Infrastructure is being deployed as code:

![1-6](./img/Lab4-Stage-1-6.png)

***

### Stage 2: Let’s put a little Sec into our DevOps pipeline:

1. We will enforce a security rule that there must not exist security groups allowing open SSH port to the internet.

![2-1](./img/Lab4-Stage-2-1.png)

The following actions are going to be performed in the stages of our Pipeline:
1. __Source stage__: In this example, the pipeline gets the CloudFormation template from the CodeCommit repository that creates the VPC with NACL, IGW, security groups and EC2 instances.

2. __StaticCodeAnalysis stage__: This stage passes the CloudFormation template and pipeline name to a Lambda function, CFNValidateLambda. This function performs the static code analysis. It uses the regular expression language to find patterns and identify security group policy violations. If it finds violations, then Lambda fails the pipeline and includes the violation details.
Here is the regular expression that Lambda function using for static code analysis of the open SSH port:

```console
"^.*Ingress.*(([fF]rom[pP]ort|[tT]o[pP]ort).\s*:\s*u?.(22).*[cC]idr[iI]p.\s*:\s*u?.((0\.){3}0\/0)|[cC]idr[iI]p.\s*:\s*u?.((0\.){3}0\/0).*([fF]rom[pP]ort|[tT]o[pP]ort).\s*:\s*u?.(22))"
```

2. __CreateEnv stage__: After the static code analysis is completed successfully, the pipeline creates the stack resources with the CloudFormation template.

3. __DynamicStackValidation stage__: This step triggers the StackValidationLambda Lambda function. It passes the stack name and pipeline name in the event parameters. Lambda validates the security group deployed for the following security controls. If it finds violations, then Lambda deletes the stack, stops the pipeline, and returns an error message.

The following is the sample Python code used by AWS Lambda to check if the SSH port is open to the approved IP CIDR range (in this example, 72.21.196.67/32):
```python
for n in regions:
    client = boto3.client('ec2', region_name=n)
    response = client.describe_security_groups(
        Filters=[{'Name': 'tag:aws:cloudformation:stack-name', 'Values': [stackName]}])
    for m in response['SecurityGroups']:
        if "72.21.196.67/32" not in str(m['IpPermissions']):
            for o in m['IpPermissions']:
                try:
                    if int(o['FromPort']) <= 22 <= int(o['ToPort']):
                        result = False
                        failReason = "Found Security Group with port 22 open to the wrong source IP range"
                        offenders.append(str(m['GroupId']))
                except:
                    if str(o['IpProtocol']) == "-1":
                        result = False
                        failReason = "Found Security Group with port 22 open to the wrong source IP range"
                        offenders.append(str(n) + " : " + str(m['GroupId']))
```

The next stages remain the same, building, deploying the code to dev and prod environments (with a manual approval action).

***

### Stage 3: Deploy the resources:
1. Create a S3 bucket and upload the zip file for our Lambda functions:
```console
aws s3 mb <YOUR-INITIALS>-$accountId-$region
aws s3 cp ./codepipeline-lambda.zip s3://<YOUR-INITIALS>-$accountId-$region
```

2. Create the resources for our DevSecOps stages in the pipeline:
```console
aws cloudformation create-stack lab4-resources --template-body file://lab4-resources.json --capabilities CAPABILITY_IAM --parameters ParameterKey=S3Bucket,ParameterValue=<YOUR-INITIALS>-$accountId-$region --region $region
```

After the template deployment is complete, go to the CodePipeline console.

Add the following stages and actions in the pipeline:

__Add stage to Pipeline after Source and before CreateEnv__

Stage Name: `StaticCodeAnalysis`

Action name: `CFNParsing`
lambda

__Add stage to Pipeline after CreateEnv and before Build__

Stage Name: `DynamicStackValidation`

Action name: `StackValidation`
lambda

IMG

Click on the “Release” button again to check the security validations.

See that our pipeline has failed. Click on “Details” under the action “StaticCodeAnalysis” of the “CFNParsing” stage and then click on “Link to execution details”.

IMG

The CloudWatch logs associated to our Lambda function will be opened.
Select the newest Log Stream and go to the bottom of the log:

IMG

We can see that the rules “IngressOpenToWorld” and “SSHOpenToWorld” matched, causing the pipeline to fail and the risk value associated.


Verify the Lambda functions for seeing the security validation logic. Note that the first Lambda function checks DynamoDB for the security rules.

Go to the DynamoDB console and check the “DDBRules”. See how rules, categories and risk values could be used to determine when to deploy to next stage or not. Security team could specify rules once and the automation would take care of the rest.

IMG-DDB

### Stage 4: Congratulations, you have finished the workshop!
You have learned how to create your own secure, CI/CD pipeline with AWS tools like CodePipeline, CodeCommit, CodeBuild, CodeDeploy, CloudFormation, Lambda.

[Check the this AWS blog post for more details on DevSecOps](https://aws.amazon.com/blogs/devops/implementing-devsecops-using-aws-codepipeline/).


You can now proceed to cleanup all the resources

[Cleanup](README.md#clean-up)