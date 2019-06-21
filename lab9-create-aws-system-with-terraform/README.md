# Lab 9: Create AWS system with Terraform
In this lab you will build an entire AWS system with terraform

## Task 1: Install Terraform
In this task you will install Terraform

1. Set your region to us-east-1 (N. Virginia, everyone can use this region for this lab) and create an EC2 instance (name the instance firstname-lastname-lab9-host, and create and download a key pair named firstname-lastname-l8-key) and SSH into it, then execute the commands: `wget https://releases.hashicorp.com/terraform/0.12.2/terraform_0.12.2_linux_amd64.zip` and `sudo unzip terraform_0.12.2_linux_amd64.zip -d /usr/bin/`.

Now Terraform is installed.

2. Execute the command `nano` and paste in the content below (replace firstname-lastname with your firstname and lastname):

```tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "firstname-lastname-lab9" {
  ami = "ami-2d39803a"
  instance_type = "t2.micro"
}
```

This is a very simple configuration to create a basic EC2 instance

3. Type Ctrl-O and save the file as main.tf and then type Ctrl-X to exit

4. First create a new access key and secret key and execute some commands to set them in environment variables.  Execute the commands: `export AWS_ACCESS_KEY_ID=MYACCESSKEYID` and `export AWS_SECRET_ACCESS_KEY=MYSECRETACCESSKEY` (replace the values with your access key values).  Then let's go ahead and create it with the commands:

```bash
terraform init
terraform apply
``` 

Once the output indicates the resources have been created you can view it in the Services->EC2 section
(make sure you have set your region to N.Virginia in the top right region selector, N.Virgina is us-east-1).

## Task 2: Update the instance (add a name)

1. When you view the instance, notice that it doesn't currently have a name.  That is easy to fix, update the 
configuration to (using `nano main.tf`):

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "firstname-lastname-lab9" {
  ami = "ami-2d39803a"
  instance_type = "t2.micro"
  tags = {
    Name = "firstname-lastname-l9-ws"
  }
}
```

Great, now if you view the instance again you should see it's name appears as you've entered it (feel free to experiment with renaming it, but change it back when you're done).

## Task 3: Give the instance a simple start-up script to provide a basic web-server

1. Next, let's set a start-up script using the user_data attribute (For background see: [User Data description](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html), [user_data attribute](https://www.terraform.io/docs/providers/aws/r/instance.html#user_data)).  We'll use the heredoc string format to allow us the have the string span multiple lines ([heredoc strings](https://www.terraform.io/docs/configuration-0-11/variables.html#strings)).  Update main.tf with nano: `nano main.tf`

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "firstname-lastname-lab9" {
  ami = "ami-2d39803a"
  instance_type = "t2.micro"

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF

  tags = {
    Name = "firstname-lastname-l9-ws"
  }
}
```

## Task 4: Set up a security group to allow external traffic to reach the instance

1. Before we'll be able to actually connect to the instance via HTTP we'll need to add a [Security group](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html).  The resource for that is [aws_security_group](https://www.terraform.io/docs/providers/aws/r/security_group.html).  Update main.tf with nano: `nano main.tf`

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "firstname-lastname-lab9" {
  ami = "ami-2d39803a"
  instance_type = "t2.micro"

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF

  tags = {
    Name = "firstname-lastname-l9-ws"
  }
}

resource "aws_security_group" "firstname-lastname-lab9" {
  name = "firstname-lastname-ws-sg"
  ingress {
    from_port = 8080
    to_port = 8080
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
Notice we're allowing connection on 8080 via tcp and using the source [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) 0.0.0.0/0 (corresponds to allowing traffic on tcp:8080 from all possible source IP addresses)

## Task 5: Minimize duplication (and increase modularity/flexibility) by factoring out variable

1. Create a variables.tf file (with `nano`) and add the following variable

```hcl
variable "server_port" {
  description = "The port the server will use for HTTP requests"
  default = 8080
}
```

Then update the other places the 8080 is being used in **main.tf** and replace with **var.server_port** (for non-strings) and **"${var.server_port}"** 

## Task 6: Update instance to use security group

1. Now that we've created the security group we need to update the EC2 instance to use it via the attribute [vpc_security_group_ids](https://www.terraform.io/docs/providers/aws/r/instance.html#vpc_security_group_ids)

Add the attribute to the EC2 instance resource

```hcl
vpc_security_group_ids = [aws_security_group.firstname-lastname-lab9.id]
```

Remember to change firstname-lastname to your firstname and lastname

Recall that this means Terraform will create the security group first before the EC2 instance (because of the implicit dependency, [see here](https://www.terraform.io/docs/internals/graph.html)).

2. Ok, now let's test it out
```bash
terraform apply
```

Once the terraform apply completes, then look up the public IPv4 address of your EC2 instance and put it in your browser bar followed by :8080.  You should see "Hello, World" (the web-server is up!)

3. Rather than having to look up the public IP address, we can have it printed out to the screen as an output.  Create a new file called outputs.tf and place the following in it (Use `nano`)

```hcl
output "public_ip" {
  value = aws_instance.mark-pevec-lab9.public_ip
}
```

4. You can test it out by doing another apply (it won't create new resources because they already exist, but will show the output):

```bash
terraform apply
```

## Task 7: Scale out

In the real-world typically you don't deploy a single web-server but have a set of them to handle more traffic and potential server failures.  Let's do that now.

1. To accomplish that we'll create an [Auto-scaling group (ASG)](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html), which requires a [launch configuration](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchConfiguration.html).  We can create these via the AWS Provider resources [aws_launch_configuration](https://www.terraform.io/docs/providers/aws/r/launch_configuration.html) and [aws_autoscaling_group](https://www.terraform.io/docs/providers/aws/r/autoscaling_group.html)

First add the launch configuration (to main.tf):

```hcl
resource "aws_launch_configuration" "firstname-lastname-lab9" {
  image_id = "ami-2d39803a"
  instance_type = "t2.micro"
  security_groups = [aws_security_group.firstname-lastname-lab9.id]
  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p ${var.server_port} &
              EOF
  lifecycle {
    create_before_destroy = true
  }
}
```

Notice it looks exactly like the aws_instance config (that's because it is a template and so has the same attributes).  We did add an extra [lifecycle](https://www.terraform.io/docs/configuration/resources.html#lifecycle-lifecycle-customizations) which is required for launch configurations used with ASGs (but is optional for EC2 instances).  This particular lifecycle ensures new resources are created before old ones are destroyed when swapping.  When this particular lifecycle value is set on a resource, it also has to be set on each resource it depends on and so we'll need to set it on the security group as well.

Update the aws_security_group to include:

```hcl
lifecycle {
  create_before_destroy = true
}
```

2. Now create the ASG:

```hcl
data "aws_availability_zones" "all" {}

resource "aws_autoscaling_group" "firstname-lastname-lab9" {
  launch_configuration = aws_launch_configuration.firstname-lastname-lab9.id
  availability_zones = data.aws_availability_zones.all.names
  min_size = 2
  max_size = 10
  tag {
    key = "Name"
    value = "firstname-lastname-lab9-asg"
    propagate_at_launch = true
  }
}
```

The ACG will autoscale between 2-10 instances based on traffic and will spread them out over all [availability zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html).  To save from having to hard-code the list of possible availability zones we use the data source [aws_availability_zones](https://www.terraform.io/docs/providers/aws/d/availability_zones.html)

## Task 8: Add a load-balancer

1. Now we have a bunch of instances but we still need something in front of them with a single IP address that can distribute the traffic received to that single IP address evenly among them.  That is a Load Balanacer.  There are several types but we'll use an [Elastic Load Balancer (ELB)](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html). We can create that via an [aws_elb](https://www.terraform.io/docs/providers/aws/r/elb.html)

```hcl
resource "aws_elb" "firstname-lastname-lab9" {
  name = "firstname-lastname-lab9-elb"
  availability_zones = data.aws_availability_zones.all.names
  listener {
    lb_port = 80
    lb_protocol = "http"
    instance_port = var.server_port
    instance_protocol = "http"
  }
}
```

2. We set up our ELB to listen on port 80 and forward the traffic to our ASG on the ports they expect (var.server_port).  We'll need to allow that traffic via a security group though so we add:

```hcl
resource "aws_security_group" "firstname-lastname-lab9-elb" {
  name = "firstname-lastname-elb-sg"
  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

and then we need to add it to our aws_elb resource:

```hcl
security_groups = [aws_security_group.firstname-lastname-lab9-elb.id]
```

3. You can also add a health-check from the ELB to the instances in the ASG where it pings them periodically and if they don't respond they are considered unhealthy and the ELB won't route traffic to them

add the following to the aws_elb resource:

```hcl
health_check {
  healthy_threshold = 2
  unhealthy_threshold = 2
  timeout = 3
  interval = 30
  target = "HTTP:${var.server_port}/"
}
```

To allow the health-check traffic we need to update the aws_security_group for the ELB (simplest is to allow any outbound traffic):

```hcl
egress {
  from_port = 0
  to_port = 0
  protocol = "-1"
  cidr_blocks = ["0.0.0.0/0"]
}
```

4. Now we need to update the ASG to register instances with the ELB (so the ELB knows who to health-check).  We can also update the ASG to use the same health-checker as the ELB and to try restarting instances that aren't healthy.  Add the following to the config for the aws_autoscaling_group

```hcl
load_balancers = [aws_elb.firstname-lastname-lab9.name]
health_check_type = "ELB"
```

Finally, let's add an output to give us the DNS name generated for the ELB so we can test.  Add the following output to the outputs.tf file

```hcl
output "elb_dns_name" {
  value = aws_elb.firstname-lastname-lab9.dns_name
}
```

Now to test do a
```bash
terraform apply
```

Copy the URL fo the elb_dns_name and paste in a browser.  You should see "Hello, World" and the requests are being load balanced among the instances.  You can go to the EC2 instances page and try terminating an instance to see what happens (a new one will be created).

When you're finished do a
```bash
terraform destroy
``` 
to clean up and remove the resources.