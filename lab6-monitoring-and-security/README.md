# Lab 6: Monitoring and Security

This lab provides practice with using CloudWatch metrics, events, and logs

## Task 1: Create instance and install the CloudWatch agent
This task will walk you through installing the CloudWatch agent on an instance via the Systems Manager

1. In the **Services->EC2** page, click the **Launch Instance** button, and click the **Select** button for the AMI image at the top of the list

2. Leave the default setting for the machine type and click the **Next: Configure Instance Details** button in the bottom right

3. Set the **IAM Role** to **SystemsManagerRole**.  This will allow the Systems Manager to manage the instance.  Click on **Advanced Details** and add the below script into the **UserData** textbox

```bash
#!/bin/bash
# Install Apache Web Server
yum install -y httpd

# Turn on web server
systemctl enable httpd.service
systemctl start  httpd.service

# Download App files
wget https://us-west-2-tcprod.s3.amazonaws.com/courses/ILT-TF-100-SYSOPS/v3.3.6/lab-2-ec2-linux/scripts/dashboard-app.zip
unzip dashboard-app.zip -d /var/www/html/
```

4. Create a new security group and add a rule to allow traffic from 0.0.0.0/0 on port 80, add a tag with key=Name and value set to firstname-lastname-lab6 (replace firstname-lastname with your firstname and lastname) and then **Review and Launch**, select **Proceed without a key pair** and click on the checkbox and then on **Launch**

5. In the **Services->Systems Manager** click on **Run command** in the left-hand pane then the **Run command** orange button, then select **AWS-ConfigureAWSPackage**, then in the **Command Parameters** section, enter **AmazonCloudWatchAgent** for the **Name** and **latest** for the **Version**, in the **Targets** section click the checkbox next to the instance you just created and then click **Run**

This installs the CloudWatch agent which will then start sending logs and metrics data from in the instance into CloudWatch

6.  In the left-hand pane click on **Parameter Store**, then click **Create parameter**, and enter firstname-lastname-monitor in the **Name** field and the below JSON in the **Value** field (replace firstname-lastname with your firstname and lastname):

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "log_group_name": "firstname-lastname-access-log",
            "file_path": "/var/log/httpd/access_log",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%b %d %H:%M:%S"
          },
          {
            "log_group_name": "firstname-lastname-error-log",
            "file_path": "/var/log/httpd/error_log",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%b %d %H:%M:%S"
          }
        ]
      }
    }
  },
  "metrics": {
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_idle",
          "cpu_usage_iowait",
          "cpu_usage_user",
          "cpu_usage_system"
        ],
        "metrics_collection_interval": 10,
        "totalcpu": false
      },
      "disk": {
        "measurement": [
          "used_percent",
          "inodes_free"
        ],
        "metrics_collection_interval": 10,
        "resources": [
          "*"
        ]
      },
      "diskio": {
        "measurement": [
          "io_time"
        ],
        "metrics_collection_interval": 10,
        "resources": [
          "*"
        ]
      },
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 10
      },
      "swap": {
        "measurement": [
          "swap_used_percent"
        ],
        "metrics_collection_interval": 10
      }
    }
  }
}
```

7. Then click on **Create parameter**

This is the configuration for the CloudWatch agent and is configuring some log files to collect as well as some metrics.

8. In the left-hand pane click on **Run commnand**, then click the **Run command** button and enter **AmazonCloudWatch-ManageAgent** into the search bar, then select it, then in the **Command parameters** section in the **Optional Configuration Location** field enter firstname-lastname-monitor (replace firstname and lastname), in the **Targets** section select your created instance and click **Run**

This will start up the CloudWatch agent and have it take the configuration from the Parameter store parameter you specified previously

## Task 2: Monitor application logs in CloudWatch logs
Here you will trigger some log generation and then view those logs in CloudWatch logs

1. Visit the web-server in the instance you created by pasting its public IP address in the browser (you should see the dashboard web-app),  then add **/start** to the end of the path and hit return, you should get a **Not Found** error

Now we can check the CloudWatch logs to verify we can see that error visible there

2. In **Services->CloudWatch** (in the **Management and Governance** section), click on the **Logs** section in the left-hand pane, and then click on the **firstname-lastname-access** (where firstname-lastname is your firstname and lastname) in the **Logs Groups** list, then click on the only entry in the list,  you should see the web-server access logs and a line with **GET /start HTTP/1.1" 404**

3. Return to the previous page (by clicking on **Logs** in the left hand pane) and select the **firstname-lastname-access** (where firstname-lastname is your firstname and lastname, click the radio button to the left of it) and then click **Create metric filter**, and paste **[ip, id, user, timestamp, request, status_code=404, size]** into the **Filter pattern** field, then click **Test pattern**, and in the **Results** section click **Show test results**

This searches the log for lines matching the filter

4. Click **Assign metric**, then in the **Metric Name** field enter firstname-lastname-metric-filter and click **Create Filter**

5. Click **Create alarm** (it is in the box below and to the right of the **Add Metric Filter** button), and enter **5** in the **than** field (leave the condition set to **Greater**), then click **Next**, then select the **Create a new topic**, enter firstname-lastname-topic in the field below **Create a new topic...**, and enter your email in the field below **Email endpoints that will receive the notification…** and click the **Create topic** button, then click **Next**

6. You should receive an email to the address you specified that has a **Confirm subscription** link, click on it

7. Enter firstname-lastname-404 in the field below **Define a unique name** and click **Next**, then click **Edit** at the top of the page and set **Period** to **1 minute**, click **Next** until you see a **Create alarm** button at the bottom and then click it

8. You should come to a page with a list of alarms that includes the one you just created.

9. Return to the browser that you used to trigger the **Not Found** page and refresh it more than 5 times

10. Wait a few minutes and you should see an **Alarm** triggered in red on the CloudWatch page and you should receive an email indicating an alarm

## Task 3: Monitor instance metrics
In this task you will view the metrics from your instance

1. In **Services->EC2** in the left-hand pane click **Instances** and then select your instance (via the checkbox to the left) and then click on the **Monitoring** tab in the pane below, examine the charts and click on a few to see more details, click on the **View all CloudWatch metrics** link above the charts

2. Click the **CWAgent** link, then click **host**, you will see metrics related to system memory, when you click on metrics (in the checkbox to the left of them) you will see those metrics added to the chart above

## Task 4: Create real time notifications
In this task you will create CloudWatch events for real-time notification of when an instance is stopped or terminated

1. In the **Services->CloudWatch** page click on **Rules** in the left-hand pane and click the **Create rule** button, in the **Event source** section for **Service Name** select **EC2**, and for **Event Type** select **EC2 Instance State-change Notification**, then select **Specific state(s)** and in the dropdown select **stopped** and **terminated**

2. Click **Add Target**, then click on the dropdown that is set to **Lambda function** and change it to **SNS Topic** and select the topic you created earlier firstname-lastname-topic

3. Click **Configure details** at the bottom right of the page, in the **Rule definition** section, set **Name** to firstname-lastname-rule, then click **Create rule**.

This will now trigger sending messages to your SNS topic whenever the rule event happens, and you have already added a subscription to receive an email when a message is place on that topic, so now when the instance state changes to stopped or terminated you should get an email.

4. In the **Services->EC2** page, click on the **Instances** in the left-hand pane, select the instance you created via the checkbox to the left of it, then click **Actions** then select **Instance state->Stop**.  Verify that within a few minutes you receive an email telling you the instance state has moved to stopped