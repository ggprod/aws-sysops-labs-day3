# Lab 7: Managing Resources with Tagging
In this lab you will see how tags added to instances can aid in manging them

## Task 1: Create several instances with the same tag
In this task you will create three instances each with the same tag and one other one that will manage them

1. Create 3 instances with default values with the names (via Tag with key=Name and corresponding value) firstname-lastname-lab7-1, firstname-lastname-lab7-2, firstname-lastname-lab7-3.  Launch all 3 without a key pair

2. Add another tag to all 3 instances with key=firstname-lastname-lab7-tag and value set to stop-me

3. Create a 4th instance with default values and name firstname-lastname-lab7-host and create and download a key pair for that instance (call it firstname-lastname-l7-key)

4. SSH into firstname-lastname-lab7-host, create a new access key and secret key in IAM, and the run `aws configure` (set the access key detaills to the new one, and the region to be the same one that you created all the instances in) and run the following command to list out all instances with the stop-me tag: `aws ec2 describe-instances --filter "Name=tag:firstname-lastname-lab7-tag,Values=stop-me"` (remember to swap out firstname-lastname with your firstname and lastname)

The response you receive should include the details of the 3 instances you created with that tag

## Task 2: Create and run a script that stops all 3 instances with the tag
In this task you will create and run a python script that stops all the instances with the tag you created

1. In your SSH session to firstname-lastname-lab7-host, run the command `nano`

2. Copy and past the below python script into the nano editor and update the firstname-latname to be your firstname and lastname:

```python
#!/usr/bin/env python

import boto3

# Connect to the Amazon EC2 service
ec2 = boto3.resource('ec2')

# Loop through each instance
for instance in ec2.instances.all():
  state = instance.state['Name']
  if instance.tags is not None
    for tag in instance.tags:

      # Check for the 'stopinator' tag
      if tag['Key'] == 'firstname-lastname-lab7-tag':
        action = tag['Value'].lower()
        
        # Stop?
        if action == 'stop-me' and state == 'running':
          print "Stopping instance", instance.id
          instance.stop()
```

See if you can read the script, it is running through all the instances in the default region and checking to see if they have tags, and if they do, checking if they have a tag matching firstname-lastname-lab7-tag and if they do and the value is stop-me and they are in the running state then it is stopping those instances

3. Type Ctrl-O to save the file, give it the name stopinator.py and hit return, then Ctrl-X to exit nano

4. Execute the following commands to download and install pip (python package manager): `sudo curl -O https://bootstrap.pypa.io/get-pip.py` then `sudo python get-pip.py`, then execute the following command to install boto3: `sudo pip install boto3`

5. Execute the script via the command `python stopinator.py`

You should see output indicating the script has stopped 3 instances

6. In the **Services->EC2** page, in the left-hand pane, click on **Instances** and verify your 3 instances have been stopped