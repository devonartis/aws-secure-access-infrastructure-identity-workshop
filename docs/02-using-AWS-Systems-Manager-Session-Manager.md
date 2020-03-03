# Module 2: Session Manager ~ 30 minutes

We were given the task to setup secure administrative access to our instances but we know that anytime someone logs into a server changes can be made and those changes could inadvertently affect the security of the host. Instead of hoping our system security doesn't change, let's change the access pattern and restrict all access into production systems. This approach to administrative access is called *Immutable Infrastructure*, which includes managing services and software deployments on IT resources wherein components are replaced rather than changed. An application or service is effectively redeployed each time changes are required. Given this is a new access pattern for our team, we will continue to allow access to our development systems. Using Amazon Systems Manager Session Manager and AWS IAM with tag based permissions we will implement this exact scenario and review the audit logs.

In this Module we will create IAM roles with permissions to enable Session Manager access. Additionally, we are going to deploy 4 systems:

 * AWS EC2 Instance tagged with development
 * AWS EC2 Instance tagged with production
 * Mock on-premises system tagged with development
 * Mock on-premises system tagged with production

Please note our mock on-premises systems will be EC2 instances, however we will treat them like on-premises systems by not assigning an Instance Profile.

!!! Terminology
    A *managed instance* is any machine configured for AWS Systems Manager. You can configure Amazon EC2 instances or on-premisess machines in a hybrid environment as a *managed instance*. Systems Manager supports various distributions of Linux, including Raspberry Pi devices, and Microsoft Windows Server. In the AWS Management Console, any machine prefixed with "mi-" is an on-premisess server or virtual machine (VM) *managed instance*. AWS Systems Manager offers a standard-instances tier and an advanced-instances tier for servers and VMs in your hybrid environment. Advanced instances also enable you to connect to your hybrid machines by using AWS Systems Manager Session Manager. Session Manager provides interactive shell access to your instances. While this lab focuses on Session Manager which is a capability of AWS Systems Manager there are several other capabilities of AWS Systems Manager that you can take advantage of, more information about <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/features.html" target="_blank">System Manager Capabilities</a>

## Tasks

1. Create IAM roles and permissions to enable Session Manager
2. Setup AWS Systems Manager to manage systems on-premises
3. Create instances and install the SSM Agent
4. Configure Systems Manager and enable management of on-premises systems
5. Create IAM users and policy restrictions based on tags
6. Configure logging
7. Confirm appropriate access and review logs

### Task 1: Create IAM roles and permissions to enable Session Manager for Amazon EC2 Instances

1. Navigate to the <a href="https://us-east-1.console.aws.amazon.com/cloud9/home" target="_blank">AWS Cloud9</a> console.

2. On the left side of the Cloud9 Console is a menu, Click on **Account environments** and select **Open IDE** on the environment that has been deployed for you. Perform the following commands in the same terminal window that you setup your credentials.

3. Create an instance profile:

```bash
aws iam create-instance-profile --instance-profile-name SSMLabProfile
```

4. Create the json trust policy doc to attach to the IAM role. Create a new file with the following contents, save the file name: **lab-role-trust-policy.json**:
```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }
}
```

5. Create an IAM role using the trust policy above:
```bash
aws iam create-role --role-name SSMLabRole --assume-role-policy-document file://lab-role-trust-policy.json
```

6. Add the role to the instance profile:
```bash
aws iam add-role-to-instance-profile --role-name SSMLabRole --instance-profile-name SSMLabProfile
```

7. Attach the existing **EC2RoleSSMAccess** to the newly created instance-profile:
```bash
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforSSM --role-name SSMLabRole
```

8. Attach the instance profile to the system.

9. Create the json trust policy doc to attach to the IAM role. Create a new file with the following contents, save the file name: ssmservice-trust-policy.json
```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {"Service": "ssm.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }
}
```

10. Create a new role named **SSMServiceRole**:
```bash
aws iam create-role --role-name SSMServiceRole --assume-role-policy-document file://ssmservice-trust-policy.json
```

11. As a starting point, we will use AmazonSSMManagedInstanceCore grant permission for Systems Manager to interact with your instances. AmazonSSMManagedInstanceCore, enables an instance to use AWS Systems Manager service core functionality. Depending on your operations plan, you might need permissions represented in one or more of the other three policies. To view this policy in the console go to services, IAM, Access Management, Policies and search for the **AmazonSSMManagedInstanceCore** policy, click on the policy, view the permissions by clicking on the permissions tab. Now attach this policy to the role using attach-role-policy command.

```bash
aws iam attach-role-policy --role-name SSMServiceRole --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

12. Let's make sure we allow our on-premises systems to write to CloudWatchLogs so that we can use the same Audit and Logging tools to review administrative access. We'll use the same Attach-role-policy to attach a policy that enables the SSMServiceRole to write to CloudWatchLogs.
```bash
aws iam attach-role-policy --role-name SSMServiceRole --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

### Task 2: Configure Session Manager to manage systems on-premises

The activation process provides an Activation Code and ID which functions like an access key ID and secret key to provide secure access to the Systems Manager service form your *managed instances*.

!!! Attention
    If you've been using the Cloud9IDE in this lab please continue to run these commands from there.

1.Open the command line, Create a managed-instance activation for your Dev instance, copy the "activation-code" and "activation-id" and label it as you DevOnPrem in your scratch pad, we will use it later.
``` bash
aws ssm create-activation --default-instance-name DevOnPrem --iam-role SSMServiceRole --registration-limit 10 --region us-east-1
```

2.Create a managed-instance activation for your Prod instance, copy the "activation-code" and "activation-id" and label it as you ProdOnPrem in your scratch pad, we will use it later.
``` bash
aws ssm create-activation --default-instance-name ProdOnPrem --iam-role SSMServiceRole --registration-limit 10 --region us-east-1
```

**Store the managed-instance Activation Code and Activation ID in a safe place.** You specify this Code and ID when you install SSM Agent on systems on-premises. The code and ID combination functions like an Amazon EC2 access key ID and secret key to provide secure access to the Systems Manager service from your *managed instances*.

>If you lose the Code and ID, you must create a new activation. An activation expiration is a window of time when you can register on-premisess machines with Systems Manager, default is 24 hours. An expired activation has no impact on your servers or virtual machines (VMs) that you registered with Systems Manager. This means that if an activation expires then you can’t register more servers or VMs with Systems Manager by using that specific activation. You simply need to create a new one. All of the servers and VMs that you registered will continue to be registered Systems Manager *managed instances* until you remove or disable SSM Agent on the server or VM and thereby unregister it. <a href="https://docs.aws.amazon.com/cli/latest/reference/ssm/create-activation.html" target="_blank">For more details:</a>

### Task 2: Create instances and install the SSM Agent

Please note that in order to create a mock on-premises system we will be creating and using a key-pair and not using them for the AWS EC2 instances. We do not need a key pair for the AWS EC2 instances because we are using Session Manager to gain direct access.

!!! Attention
    If you are using Cloud9, please perform the following on the Cloud9 instance.

1. Create a key pair for SSH access to the on-premises systems. Follow the instructions for **Creating a Key Pair Using Amazon EC2**<a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair"_blank"> here.</a> Copy the name of your key pair to your scratch pad.

!!! info
    What is a Key Pair?

    Amazon EC2 uses public key cryptography to encrypt and decrypt login information. Public key cryptography uses a public key to encrypt a piece of data, and then the recipient uses the private key to decrypt the data. The public and private keys are known as a key pair. Public key cryptography enables you to securely access your instances using a private key instead of a password.

    When you launch an instance, you specify the key pair. You can specify an existing key pair or a new key pair that you create at launch. At boot time, the public key content is placed on the instance in an entry within ~/.ssh/authorized_keys. To log in to your instance, you must specify the private key when you connect to the instance.

2.Go back to the CloudFormation Stack, find the stack that starts with "aws-cloud9-" click on **Resources**, copy the ID next to the **InstanceSecurityGroup** to a scratch pad. Now find the stack named "InfrastructureIdentity-Env-Setup", click on **Resources**, copy the ID next to **PublicSubnet1** to the same scratch pad, you will need them for the following steps.

!!! Tip
    At this point, make sure you have captured 3 items on your scratch pad:
    > 1. The security group ID of the InstanceSecurityGroup
    > 2. The subnet ID of PublicSubnet1
    > 3. The name of the key pair you created.

3.Build the production instance using the following cli command:
>**Note**: Replace "subnet-xx" with the PublicSubnet1 ID and replace "sg-xx" with the InstanceSecurityGroup ID from your scratch pad.
```bash
aws ec2 run-instances --iam-instance-profile Name=SSMLabProfile --image-id ami-0080e4c5bc078760e --instance-type t1.micro --subnet-id "subnet-xx" --security-group-ids "sg-xx" --region us-east-1 --tag-specifications 'ResourceType=instance,Tags=[{Key="Name",Value="ProdEC2Instance"},{Key="Environment",Value="Prod"}]'
```

4.Build the development instance using the following cli command:  
>**Note:** Replace "subnet-xx" with the PublicSubnet1 ID and replace "sg-xx" with the InstanceSecurityGroup ID from your scratch pad.
```bash
aws ec2 run-instances --iam-instance-profile Name=SSMLabProfile --image-id ami-0080e4c5bc078760e --instance-type t1.micro --subnet-id "subnet-xx" --security-group-ids "sg-xx" --region us-east-1 --tag-specifications 'ResourceType=instance,Tags=[{Key="Name",Value="DevEC2Instance"},{Key="Environment",Value="Dev"}]'
```

5.Build a MOCK production on-premises instance using the following cli command:
>**IMPORTANT:** We are not assigning an Instance Profile as these instances will mock our on-premises servers.

>**Note:** Replace "subnet-xx" with the PublicSubnet1 ID, "sg-xx" with the InstanceSecurityGroup ID and "keyname-xx" with the key pair name from your scratch pad.

```bash
aws ec2 run-instances --image-id ami-0080e4c5bc078760e --instance-type t1.micro --subnet-id "subnet-xx" --security-group-ids "sg-xx" --region us-east-1 --key-name "keyname-xx" --tag-specifications 'ResourceType=instance,Tags=[{Key="Name",Value="ProdOnPrem"},{Key="Environment",Value="Prod"}]'
```

6.Build the MOCK on-premises development instance using the following cli command:
>**IMPORTANT:** We are not assigning an Instance Profile as these instances will mock our hybrid servers. **Please use your own key-pair**.

>**Note:** Replace "subnet-xx" with the PublicSubnet1 ID, "sg-xx" with the InstanceSecurityGroup ID and "keyname-xx" with the key pair name from your scratch pad.

```bash
aws ec2 run-instances --image-id ami-0080e4c5bc078760e --instance-type t1.micro --subnet-id "subnet-xx" --security-group-ids "sg-xx" --region us-east-1 --key-name "keyname-xx" --tag-specifications 'ResourceType=instance,Tags=[{Key="Name",Value="DevOnPrem"},{Key="Environment",Value="Dev"}]'
```

8.From your Cloud9 instance use the ssh command to connect to DevOnPrem instance. You specify the path and file name of the private key (.pem), the user name for your AMI, and the private ip address for the instance. Additionally, you will need the private IPv4 address for your instance using the Amazon EC2 console. From the Amazon EC2 console check the box next to the DevOnPrem instance, a menu with details including the Private IPs will show on the bottom of the screen, copy the Private IP address associated to your DevOnPrem instance and use it below.

>Note: Replace my-key-pair.pem with the name of the key-pair you created, this should be in your scratch pad. And replace 172.16.x.x with the Private IP associated with the DevOnPrem instance.
```bash
ssh -i /path/my-key-pair.pem ec2-user@172.16.x.x
```

9. Then run the following commands to install SSM.
> Note: Replace "activation-code" and "activation-id" for the DevOnPrem instance with the code and id on your scratch pad.
```bash
sudo yum update -Y
mkdir /tmp/ssm
curl https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm -o /tmp/ssm/amazon-ssm-agent.rpm
sudo yum install -y /tmp/ssm/amazon-ssm-agent.rpm
sudo stop amazon-ssm-agent
sudo amazon-ssm-agent -register -code "activation-code" -id "activation-id" -region us-east-1
sudo start amazon-ssm-agent
```

10.From your Cloud9 instance use the ssh command to connect to ProdOnPrem instance. You specify the path and file name of the private key (.pem), the user name for your AMI, and the private ip address for the instance. Additionally, you will need the private IPv4 address for your instance using the Amazon EC2 console. From the Amazon EC2 console check the box next to the ProdOnPrem instance, a menu with details including the Private IPs will show on the bottom of the screen, copy the Private IP address associated to your OnPrem instance and use it below.


>Note: Replace my-key-pair.pem with the name of the key-pair you created, this should be in your scratch pad. And replace 172.16.x.x with the Private IP associated with the ProdOnPrem instance.
```bash
ssh -i /path/my-key-pair.pem ec2-user@172.16.x.x
```

11. Then run the following commands to install SSM.
> Note: Replace "activation-code" and "activation-id" with the code and id on your scratch pad.
```bash
sudo yum update -Y
mkdir /tmp/ssm
curl https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm -o /tmp/ssm/amazon-ssm-agent.rpm
sudo yum install -y /tmp/ssm/amazon-ssm-agent.rpm
sudo stop amazon-ssm-agent
sudo amazon-ssm-agent -register -code "activation-code" -id "activation-id" -region us-east-1
sudo start amazon-ssm-agent

exit
```
!!! Attention
    DON’T Forget the last command above, this command start the ssm agent after activation *- sudo start amazon-ssm-agent*

### Task 3: Configure Systems Manager to enable management of on-premises systems

In order to use Session Manager to connect to on-premisess systems SSM needs to be configured for advanced-instances tier. To enable the advanced-instances tier:

1.Open the **AWS Systems Manager** console

2.In the navigation pane, choose *managed instances*.

3.Choose the **Settings** tab.

4.Choose **Change account setting**.

5.Review the information in the pop-up about changing account settings, and then, if you approve, choose the option to accept and click **Change setting**. **NOTE:** The system can take several minutes to complete the process of moving all instances from the standard-instances tier to the advanced-instances tier.

6.Select *managed instances* You should now see a new *Advanced Instances* label next to the *managed instances* Heading and you can now manage on-premises systems.

### Task 4: Create IAM users and policy restrictions based on tags
**NOTE:** The following steps require full access to IAM.

1.If you started this lab using Cloud9, c0ontinue to run the following commands from your Cloud9 session, use the create-user command to create the user.
```bash
aws iam create-user --user-name MyWorkshopUser
```

2.Create your own and assign a password to the user by replacing the '######' symbols with that password. Please ensure you use a complex password.
```bash
aws iam create-login-profile --user-name MyWorkshopUser --password '######'
```

3.Now we will create and attach an IAM Custom Policy to MyWorkshopUser. Create a new file with the following contents, save the file name: SSMDevAccess.json
>This file must reside in the same directory where your CLI session is running, or you must specify the location.
```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:DescribeSessions",
                "ssm:GetConnectionStatus",
                "ssm:DescribeInstanceProperties",
                "ec2:DescribeInstances",
                "ssm:StartSession"
            ],
            "Resource": "*"
        },
        {
            "Sid": "ReadAlltheSSMThings",
            "Effect": "Allow",
            "Action": [
                "ssm:Get*",
                "ssm:Describe*",
                "ssm:List*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "SameUserTerminate",
            "Effect": "Allow",
            "Action": "ssm:TerminateSession",
            "Resource": "arn:aws:ssm:*:*:session/${aws:username}-*"
        },
        {
            "Sid": "DenySMtoProd",
            "Effect": "Deny",
            "Action": "ssm:StartSession",
            "Resource": [
                "arn:aws:ec2:*:*:instance/*"
            ],
            "Condition": {
                "StringLike": {
                    "ssm:resourceTag/Environment": "Prod"
                }
            }
        }
    ]
}
```

4.Create the IAM policy using the file you just created
```bash
aws iam create-policy --policy-name SSMDevAccess --policy-document file://SSMDevAccess.json
```

The result should return the following:
````
 {
    "Policy": {
        "PolicyName": "SSMDevAccess",
        "PolicyId": "ANPAJIFY55FF57L6ZEHKO",
        "Arn": "arn:aws:iam::123456789012:policy/SSMDevAccess",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2019-03-21T20:51:21Z",
        "UpdateDate": "2019-03-21T20:51:21Z"
    }
}
````

5.Copy the Account ID to your scratch pad, you will need it for the next instruction. You can find the AWS Account ID located in an IAM policy ARN, between "iam" and "policy", do not include the : symbols i.e. the AWS Account ID is 123456789012 given the following ARN "Arn": `arn:aws:iam::123456789012:policy/SSMDevAccess`.

6.To attach the policy, use the attach-user-policy command, and reference the environment variable that holds the policy ARN.
```bash
aws iam attach-user-policy --user-name MyWorkshopUser --policy-arn arn:aws:iam::123456789012:policy/SSMDevAccess
```

7.Let's attach a policy that allows our workshop user to view the logs in CloudWatch.
```bash
aws iam attach-user-policy --user-name MyWorkshopUser --policy-arn  arn:aws:iam::aws:policy/CloudWatchLogsReadOnlyAccess
```
8.Let's attach a policy that allows our workshop user to view the events in CloudTrail.
```bash
aws iam attach-user-policy --user-name MyWorkshopUser --policy-arn arn:aws:iam::aws:policy/CloudWatchEventsReadOnlyAccess
```

9.Verify that the policy is attached to the user by running the list-attached-user-policies command.
```bash
aws iam list-attached-user-policies --user-name MyWorkshopUser
```
The result should return the following:
```
{
    "AttachedPolicies": [
        {
            "PolicyName": "SSMDevAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/SSMDevAccess"
        }
    ]
}
```

### Task 5: Configure Logging

Session Manager provides you with options for auditing and logging session activity in your AWS account. This allows you to do the following:
>
> *	Create and store session logs for archival purposes.
> *	Generate a report showing details of every connection made to your instances using Session Manager over the past 30 days.
> * Generate notifications of session activity in your AWS account, such as Amazon Simple Notification Service (Amazon SNS) notifications.
> *	Automatically initiate another action on an AWS resource as the result of session activity, such as running an AWS Lambda function, starting an AWS CodePipeline pipeline, or running an AWS Systems Manager Run Command document.

We will log session data using Amazon CloudWatch Logs, archive session logs to Amazon S3 and generate access reports.

1. To setup Session Manager logging in Amazon CloudWatch Logs, open the <a href="https://console.aws.amazon.com/cloudwatch/" target="_blank">CloudWatch Console</a>. In the navigation pane, choose **Logs**

2. Choose Actions, Create log group.

3. Type in the following name for the log group "SSM-Logs"

4. Now open the <a href="https://console.aws.amazon.com/systems-manager/" target="_blank">AWS Systems Manager console</a>. In the navigation pane, choose **Session Manager**

2.	Select **Configure Preferences** You have several configurations options, including:
>
> * Specify Operating System user for session
> * Enable KMS encryption for active sessions
> * Write session output to an S3 bucket
> * Send session output to CloudWatch Logs

3.	Select the check box next to **CloudWatch Logs**.

4.	Select **Choose a log group name from the list** specify the CloudWatch Logs group you  created **SSM-Logs** and select **Save**. Additionally, you can view ssm-agent logs on the instance here:
>
> * /var/log/amazon/ssm/amazon-ssm-agent.log
> * /var/log/amazon/ssm/errors.log

### Task 6: Confirm appropriate access and review logs

1.Logout of the AWS Management Console session or Open a different browser. Sign into the AWS account as **MyWorkshopUser** with the password you created earlier. Go to the <a href="https://console.aws.amazon.com/systems-manager/" target="_blank">AWS Systems Manager console</a>. In the navigation pane, choose **Session Manager**

2.Click **Start session**, select the **ProdOnPrem** name, click on **start Session** and a new session windows should open for you. This demonstrates that you can manage on-premises systems with Session Manager.

3.In the Session Manager session type the following:
```bash
whoami
```
The result should return the following:
ssm-user

3.Click Terminate to end.

4.Repeat the steps above with the DevSSM instance. You should find the same results, the result should return the following:
ssm-user

5.Click **Terminate** to end.

6.Now try the ProdSSM instance, click start Session. The result should return the following:
![Session Manager Access Denied](./images/Denied.png)

7.Go to the **AWS Systems Manager**, click on **Session Manager**, click on **Session History**. Your sessions should show up as Terminated or Terminating. Once the session has terminated you will see a link to **CloudWatch Logs**.
![Session Manager Logs in CloudWatch Logs](./images/SSM-Cloudwatch.png)

Click on it to view the audit logs of your session. CloudWatch Logs will show details of the session such as commands that were run on the host. These logs can also be stored in S3 for you.
![Session Manager Logs in CloudWatch Logs](./images/CloudwatchLogs.png)

8.If you have CloudTrail enabled, you can view the Session Manager events in CloudTrail as well however you must be login as the administrator not MyWorkshopUser. Go to **CloudTrail** and find the Event name **StartSession** to view the details of your Session Manager session that CloudTrail captures. You Should see something like this:
![Session Manager Logs in CloudTrail](./images/CloudTraillog-edited.png)


## Troubleshooting

•	If your on-premisess instances are showing offline, make sure the ssm-agent was re-started after the SSM activation process.
