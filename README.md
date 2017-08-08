# Running R on AWS

This is a guide to running R code on AWS EC2 instances. This readme explains how to configuring the AWS CLI. The script files in the repository are:

*   `setup_instance` launches an instances and runs the `setup_instance_remote` to install R from source.
*   `run_script` uses [send-command](http://docs.aws.amazon.com/cli/latest/reference/ssm/send-command.html) to execute an R script on an instance. When the script is finished, you get a notification and the instance is shut down.

## Preparations (Mac)

To use these scripts you need to have certain software installed locally. After installing [Homebrew](https://brew.sh/) install and configure [AWS-CLI](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) by writing the command below in a Terminal window:

```sh
brew install awscli
```

Create an IAM user with `AmazonEC2FullAccess` and `AmazonS3FullAccess` policies through the AWS Management Console, then run `aws configure` in your console. This creates a `~/.aws/credentials` file with the following content:

```
[default]
aws_access_key_id="key ID"
aws_secret_access_key="secret"
```

Also create an `~/.aws/config` file with your preferred settings:
```
[default]
region=us-east-1
output=json
```

And remember to set the correct permissions (`chmod 600`) for these files.

To create and manage EC2 instances we need to set up a [ssh keypair](http://docs.aws.amazon.com/cli/latest/userguide/cli-ec2-keypairs.html) and a [security group](http://docs.aws.amazon.com/cli/latest/userguide/cli-ec2-sg.html):

```sh
aws ec2 create-key-pair \
    --key-name aws-cli-key \
    --query 'KeyMaterial' \
    --output text > ~/.ssh/aws-cli-key.pem
chmod 400 ~/.ssh/aws-cli-key.pem
```

You can check that your key was saved properly by running `aws ec2 describe-key-pairs`.

## Launching an EC2 instance

### Using a pre-configured AMI
You now have two options, either use a pre-existing AMI to launc an already configured EC2 instance, e.g. [one of these](http://www.louisaslett.com/RStudio_AMI/). This can by done by running:

```sh
aws ec2 run-instances \
  --image-id "<AMI>" \
  --count 1 \
  --instance-type "<type>" \
  --key-name "<keypair name>" \
  --security-groups "<security group name>"
```

where you would have to replace the strings with whatever is appropriate.

### Building your own R instance

If you instead want to build R from scratch on a clean instance, you can use the `setup_instance` script in this repo. After filling in the following options in the two script files (`setup_instance` and `setup_instance_ec2_install`), you can execute the file (`./setup_instance`) and everything should just work. Before running the code make sure you install the following pre-requisites:

```sh
brew install jq
```

*   `KEYPAIR_NAME=""` Name of your AWS EC2 keypair
*   `KEYPAIR=""` Local path to your private key
*   `INSTANCE_TYPE=""` Type of instance, e.g. "c4.large"
*   `AMZN_LINUX_AMI=""` AMI to start from. You can find the latest [here](https://aws.amazon.com/amazon-linux-ami/).
*   `INSTANCE_NAME=""` Name of instance (not needed)

You can either launch a large instance immediately, or if you have time and want to save money, use a small instance (however nano seems to run out of memory) to build everything, save your own AMI (using the web console) and then use it to launch a large instance whenever you need it.

The default configuration sets the security group so that only your current IP address can access the instance. If your address changes for any reason, you will have to run the following code to be able to access it again:

```sh
ips=`dig +short myip.opendns.com @resolver1.opendns.com`/32
aws ec2 authorize-security-group-ingress --group-name $SEC_GR_NAME --protocol tcp --port 22 --cidr $ips
aws ec2 authorize-security-group-ingress --group-name $SEC_GR_NAME --protocol tcp --port 8787 --cidr $ips
aws ec2 authorize-security-group-ingress --group-name $SEC_GR_NAME --protocol tcp --port 3838 --cidr $ips
```
# Executing code on the instance

## S3 for data storage
Since we want to shut down the instance as soon as the script is finished we need to store all the data in an [S3 bucket](http://docs.aws.amazon.com/cli/latest/userguide/using-s3-commands.html). To create an S3 bucket write:

```
aws s3 mb s3://<bucketname>
```

We then need to create a [role](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html) for our EC2 instance and assign it so that the instance can download and upload files to the S3 bucket. Easiest is to do this through the AWS management console. Create a role with the policies `AmazonS3FullAccess` and `AmazonEC2RoleforSSM`.

# Some other useful commands

Terminating the instance:
```sh
aws ec2 terminate-instances --instance-ids $IID
```

```sh
sh_command_id=$(
aws ssm send-command \
    --instance-ids $IID \
    --document-name "AWS-RunShellScript" \
    --comment "Hej" \
    --parameters commands=whoami \
    --output text \
    --query "Command.CommandId")


aws ssm send-command --instance-ids $IID --document-name "AWS-RunShellScript" --comment "Demo run shell script on Linux Instance" --parameters commands=whoami --output text --query "Command.CommandId"
```
