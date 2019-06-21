# Lab 8: Create AMI with Packer
In this lab you will use Packer to create an AMI

## Task 1: Insstall Packer
In this task you will install Packer on either your local environment or on an EC2 instance

1. Create an EC2 instance (create and download a key pair named firstname-lastname-l8-key) and SSH into it, then execute the command: `sudo unzip https://releases.hashicorp.com/packer/1.4.1/packer_1.4.1_linux_amd64.zip -d /usr/bin/`.

## Task 2: Set up Packer for run

1. Create a new access key and secret key (unless you copied the secret key for a previous one in which case you can use the older one) and execute the following commands: `export AWS_ACCESS_KEY_ID=MYACCESSKEYID` and `export AWS_SECRET_ACCESS_KEY=MYSECRETACCESSKEY` (replace the values with your access key values)

2. Execute `nano` and then copy and paste the below file:

```bash
#!/bin/bash
# Install Apache Web Server
sudo yum install -y httpd

# Turn on web server
sudo systemctl enable httpd.service
sudo systemctl start  httpd.service

# Download App files
wget https://us-west-2-tcprod.s3.amazonaws.com/courses/ILT-TF-100-SYSOPS/v3.3.6/lab-2-ec2-linux/scripts/dashboard-app.zip
sudo unzip dashboard-app.zip -d /var/www/html/
```

3.  Type Ctrl-O, return, and save the file as setup.sh, then type Ctrl-X and return

4. Execute `nano` and the copy and past the below file (but update firstname-lastname to your firstname and lastname and your-region to the same region you created your instance in), also set your-ami to the ami-id of Amazon Linux 2 in your-region (you can see that in the top entry in the list when you click the **Launch instance** button):

```json
{
    "variables": {
        "aws_access_key": "{{env `AWS_ACCESS_KEY_ID`}}",
        "aws_secret_key": "{{env `AWS_SECRET_ACCESS_KEY`}}",
        "region":         "your-region"
    },
    "builders": [
        {
            "access_key": "{{user `aws_access_key`}}",
            "ami_name": "firstname-lastname-packer-{{timestamp}}",
            "instance_type": "t2.micro",
            "region": "your-region",
            "secret_key": "{{user `aws_secret_key`}}",
            "source_ami": "your-base-ami",
            "ssh_username": "ec2-user",
            "type": "amazon-ebs"
        }
    ],
    "provisioners": [
        {
            "type": "shell",
            "script": "./setup.sh"
        }
    ]
}
```

5. Type Ctrl-O and return, and save the file as packer.json, and then type Ctrl-X

6. Execute the following command to run Packer to build the AMI from the given configuration: `packer build packer.json`

7. Once the Packer build completes, create a new instance from the AMI (you can find it under the My AMIs tab when you are selecting an AMI), then set the **User data** (you'll find it under **Advanced details** on the **Configure Instance** page) to be the below:

```bash
# Turn on web server
sudo systemctl enable httpd.service
sudo systemctl start  httpd.service
```

leave everything else set at default except make a security group that has an inbound rule for HTTP on port 80 and give the instance the name firstname-lastname-packer-ws

8. In a browser put the public IP of the instance and verify you see the Dashboard web app