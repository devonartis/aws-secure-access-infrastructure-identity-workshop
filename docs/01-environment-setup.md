# Module 1: Setup Instructions

In the first module you'll be running a CloudFormation template which will build your Cloud9 environment and then you will manually configure the rest. AWS Cloud9 is a cloud-based integrated development environment (IDE) that lets you write, run, and debug your code with just a browser. It includes a code editor, debugger, and terminal. Cloud9 comes pre-packaged with essential tools for popular programming languages and the AWS Command Line Interface (CLI) pre-installed so you don’t need to install files or configure your laptop for this workshop.


To setup your environment please expand one of the following drop-downs (depending on if you are doing this workshop at an **AWS event** or **individually**) and follow the instructions:

??? info "Click here if you are *using your own AWS account*. You will be using your computer to run the commands."

	1. Log in to your account however you would normally.
	>
	> Please note for this lab you will need the following permissions:
	>
	> * AmazonEC2FullAccess
	> * CloudWatchFullAccess
	> * AWSCloud9Administrator
	> * AWSCloudTrailReadOnlyAccess
	> * iam:*
	> * s3:*
	> * ssm:*
	> * cloudformation:*

	Region| Deploy
	------|-----
		US East 1 (Virginia) | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=Infrastructure-Identity&templateURL=https://sa-security-specialist-workshops-us-west-2.s3-us-west-2.amazonaws.com/identity-workshop/idm-infrastructure/01-environment-setup.yaml" target="_blank">![Deploy in us-east-2](./images/deploy-to-aws.png)</a>

	2. Click the **Deploy to AWS** button above.  This will automatically take you to the console to run the template.  
	3. Click **Next** under the **Create stack** section.
	4. Click **Next** under the **Specify stack details** section (the stack name will already be filled - leave the other options in Parameters at there default settings)
	5. Click **Next** under the **Advanced options** section.
	6. Finally, acknowledge that the template will create IAM resources with custom names under **Capabilities** and click **Create stack**.
	7. This will bring you back to the CloudFormation console. You can refresh the stack set to see the latest status. Before moving on, make sure the stack finally shows **CREATE_COMPLETE**.
	8. Navigate to the <a href="https://us-east-1.console.aws.amazon.com/cloud9/home" target="_blank">AWS Cloud9</a> console.
	9. Now you need to update the credentials in your Cloud9IDE to match the credentials you use on your laptop. Type `aws configure --profile default` hit enter. Hit enter until you get to the choice **Default region name** and type in `us-east-1`. Hit enter and then enter again to leave this menu.
	10. Copy your ~/.aws/credentials file from your laptop to your Cloud9IDE session or create new credentials for this lab with the appropriate permissions as referenced in the note under step 1. You should end up with something that looks like this:<br>
	[default]</br>
	AWS_ACCESS_KEY_ID=ASIA________</br>
	AWS_SECRET_ACCESS_KEY=iZoD_______________________</br>
	11. Now you can run commands from within the Cloud9 IDE using your credentials. If you open a new tab you will need to paste in the credentials again.
	12. Move on to **Module 2**.


??? info  "Click here if you're at an *AWS event* where the *Event Engine* is being used. You will be using Cloud9 to run the commands."

    <p style="font-size:20px;">
      **Step 1** : Retrieve temporary credentials from Event Engine
    </p>

	1. Navigate to the <a href="https://dashboard.eventengine.run" target="_blank">Event Engine dashboard</a>
	2. Enter your **team hash** code.
	3. Click **AWS Console**
	4. Copy the **export** commands under the **Credentials** section for the temporary credentials (you will need these in the next step.)

    <p style="font-size:20px;">
      **Step 2** : Connect to the AWS Console via Event Engine and browse to the AWS Cloud9 IDE
    </p>

	1. Click **Open Console** from the Event Engine window
	2. Navigate to the <a href="https://us-east-1.console.aws.amazon.com/cloud9/home" target="_blank">AWS Cloud9</a> console.
	2. On the left side of the Cloud9 Console is a menu, Click on **Account environments** and select **Open IDE** on the environment that has been deployed for you.
	3. Click the **gear image** icon in the upper right hand corner to open the Cloud9 Preferences. Scroll down in the settings, click on the **AWS SETTINGS** section, go to **Credentials** and click the button next to **AWS managed temporary credentials** to disable this.
	5. Now go to a Cloud9 tab and click the **plus sign** to open a new terminal window (tab title will start with the words **bash**).
	6. Type `aws configure --profile default` hit enter. Hit enter until you get to the choice **Default region name** and type in `us-east-1`. Hit enter and then enter again to leave this menu.
	7. Then create a file in the `~/.aws` directory named `credentials` and paste in the credentials you copied from the Event Engine. You will need to remove the word **export** from the start of each line. Add `[default]` before all these rows. You should end up with something that looks like this:<br>
	[default]</br>
	AWS_ACCESS_KEY_ID=ASIA________</br>
	AWS_SECRET_ACCESS_KEY=iZoD_______________________</br>
	AWS_SESSION_TOKEN=FQoG_____________________________________________</br>
	8. Now you can run commands from within the Cloud9 IDE using the temporary credentials from Event Engine. If you open a new tab you will need to paste in the credentials again.
	9. Move on to **Module 2**.

###

## Architecture Overview

Your environment is now configured and ready for next steps.  Below is a diagram to depict the initial environment.

![VPC](./images/01-IdentityInfrastructure-Env-Setup.png)

After you have successfully setup your environment, you can proceed to the next module.
