# Deploy an ECS Cluster with Terraform
We are big fans of AWS at Mesh. We often are asked to deploy applications and services to a cloud provider with for our customers, and our go to cloud provider is in our own back yard.

While we love the robustness, reliability and peace of mind that AWS brings, their developer dashboard can often leave developers scratching their heads and wondering why things are so difficult.

At Mesh, we find every opportunity we can to build automation and tooling. Tools have the potential to save developers, and therefor our customers, significant amounts of time and build efficiency. One of the places we have invested the most in our tooling is our AWS tool chain.

Earlier this year, we started working on the `Mesh` cli. This CLI is the toolkit that each of our developers has access to on a daily basis.

One of the things we are often asked to do for our customer is stand up environments for deploying docker containers. When asked to do this, we use Elastic Container Service.

Although Elastic Container Service is a fantastic cluster management system, building and deploying a cluster is not a very straight forward proposition, even if you are a seasoned AWS developer who understands how to use the AWS dashboard. Because of this, we decided to build a tool that would automate the process of spinning up, deploying, and tearing down ECS clusters. To do so, we leveraged one of our favorite DevOps tools, `Terraform`.

## Deploying an ECS Cluster

We can deploy a complete ECS cluster with one command.

```
mesh cluster deploy --name "mesh-docker-sample" --image "meshhq/sample-node-container"
```

By default this command spins up a 3 instance cluster, that is load balance via an Application Load balancer. Underneath the hood, the command is provisioning a bunch of infrastructure. Lets break it down.

## Building IAM Roles

Before deploying an ECS Cluster, we need create two different IAM roles, and then attach specific policies to each of these roles. The roles are the following:

* [ECS Container Instance IAM Role](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/instance_IAM_role.html)
* [ECS Service Scheduler IAM Role](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/service_IAM_role.html)

We can provision these roles via the Mesh CLI.

```
mesh iam build --ecs
```

Underneath the hood, the Mesh CLI is generating these roles via Terraform config files.

[ecs-instance-role.tf](https://github.com/meshhq/terraform-ecs-cluster/blob/master/iam/ecs-instance-role.tf)

```
data "aws_iam_policy_document" "ecs-instance-policy" {
    statement {
        actions = ["sts:AssumeRole"]

        principals {
            type        = "Service"
            identifiers = ["ecs.amazonaws.com"]
        }
    }
}

resource "aws_iam_role" "ecs-instance-role" {
    name                = "ecs-instance-role"
    path                = "/"
    assume_role_policy  = "${data.aws_iam_policy_document.ecs-instance-policy.json}"
}

resource "aws_iam_role_policy_attachment" "ecs-instance-role-attachment" {
    role       = "${aws_iam_role.ecs-instance-role.name}"
    policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
}

output "ecs-instance-role-id" {
  value = "${aws_security_group.mesh-vpc-security-group.id}"
}
```

[ecs-service-role.tf](https://github.com/meshhq/terraform-ecs-cluster/blob/master/iam/ecs-service-role.tf)

```
data "aws_iam_policy_document" "ecs-service-policy" {
    statement {
        actions = ["sts:AssumeRole"]

        principals {
            type        = "Service"
            identifiers = ["ec2.amazonaws.com"]
        }
    }
}

resource "aws_iam_role" "ecs-service-role" {
    name                = "ecs-service-role"
    path                = "/"
    assume_role_policy  = "${data.aws_iam_policy_document.ecs-service-policy.json}"
}

resource "aws_iam_role_policy_attachment" "ecs-service-role-attachment" {
    role       = "${aws_iam_role.ecs-service-role.name}"
    policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
}

output "ecs-service-role-arn" {
  value = "${aws_iam_role.ecs-service-role.arn}"
}
```

## Building a VPC

Next we need to provision a number of [Virtual Private Cloud (VPC)](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html) resources. This involves the following:

* [VPC](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/getting-started-ipv4.html)
* [Subnets](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html)
* [Internet Gateway](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Internet_Gateway.html)
* [Route Table](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Route_Tables.html)
* [Network ACL](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_ACLs.html)
* [Security Groups](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html)

We can provision these VPC components via the following command.

```
mesh vpc build --name "mesh"
```

Underneath the hood, the Mesh CLI is generating the following via Terraform config files.

### vpc.tf

```
resource "aws_vpc" "mesh-vpc" {
  cidr_block = "10.0.0.0/16"
  enable_dns_hostnames = "true"

  tags {
    Name = "mesh-vpc"
  }
}

output "id" {
  value = "${aws_vpc.mesh-vpc.id}"
}
```

### subnet.tf

```
resource "aws_subnet" "mesh-vpc-subnet1" {
    vpc_id     = "${aws_vpc.mesh-vpc.id}"
    cidr_block = "10.0.0.0/24"
    availability_zone = "us-east-1a"

    tags {
        Name = "mesh-vpc-subnet"
    }
}

resource "aws_subnet" "mesh-vpc-subnet2" {
    vpc_id     = "${aws_vpc.mesh-vpc.id}"
    cidr_block = "10.0.1.0/24"
    availability_zone = "us-east-1b"

    tags {
        Name = "mesh-vpc-subnet"
    }
}

output "subnet1-id" {
  value = "${aws_subnet.mesh-vpc-subnet1.id}"
}

output "subnet2-id" {
  value = "${aws_subnet.mesh-vpc-subnet2.id}"
}
```

## Deploying our ECS Cluster

Once we have our VPC setup, its time to create an ECS Cluster. In order to create an ECS cluster, we provision the following:

* [Cluster](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_clusters.html)
* [Service](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html)
* [Task Definition](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)

We can provision these ECS components via the following command.

```
mesh ecs build --name "mesh-docker"
```
Underneath the hood, the Mesh CLI is generating the following via Terraform config files.

[cluster.tf](https://github.com/meshhq/terraform-ecs-cluster/blob/master/ecs/cluster.tf)

```
variable "cluster-name" {}

resource "aws_ecs_cluster" "mesh-ecs-cluster" {
  name = "${var.cluster-name}"
}
```

[service.tf](https://github.com/meshhq/terraform-ecs-cluster/blob/master/ecs/service.tf)

```

variable "ecs-service-role-arn" {}

resource "aws_ecs_service" "mesh-ecs-service" {
  name            = "mesh-ecs-service"
  cluster         = "${aws_ecs_cluster.mesh-ecs-cluster.id}"
  task_definition = "${aws_ecs_task_definition.mesh-sample-definition.arn}"
  iam_role        = "${aws_iam_role.ecs-service-role-arn}"
  desired_count   = 1
}
```

[task-definition.tf](https://github.com/meshhq/terraform-ecs-cluster/blob/master/ecs/task-definition.tf)

```
resource "aws_ecs_task_definition" "mesh-sample-definition" {
  family                = "mesh-sample-definition"
  container_definitions = "${file("./ecs/task-definition.json")}"
}
```

## Deploying EC2 Instances

Once our ECS Cluster is configured, the last step is to deploy some actual EC2 instances into our cluster. We do this by configuring an Auto Scaling Group with an Elastic Load Balancer. This involves the following:

* [Auto Scaling Group](http://docs.aws.amazon.com/autoscaling/latest/userguide/GettingStartedTutorial.html)
* [Launch Configuration](http://docs.aws.amazon.com/autoscaling/latest/userguide/LaunchConfiguration.html)
* [Application Load Balancer](http://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)

```
mesh ec2 deploy
```

[autoscaling-group.tf](https://github.com/meshhq/terraform-ecs-cluster/blob/master/ec2/autoscaling-group.tf)

```
variable "subnet-id-1" {}
variable "subnet-id-2" {}

resource "aws_autoscaling_group" "mesh-ecs-asg" {
    name                        = "mesh-ecs-asg"
    max_size                    = 3
    min_size                    = 1
    health_check_grace_period   = 300
    health_check_type           = "ELB"
    desired_capacity            = 2
    force_delete                = true
    launch_configuration        = "${aws_launch_configuration.mesh-ecs-launch-config.name}"
    vpc_zone_identifier         = ["${var.subnet-id-1}", "${var.subnet-id-2}"]
}
```

[launch-configuration.tf](https://github.com/meshhq/terraform-ecs-cluster/blob/master/ec2/launch-configuration.tf)

```
variable "cluster-name" {}
variable "security-group-id" {}
variable "ecs-instance-role" {}

resource "aws_launch_configuration" "mesh-ecs-launch-config" {
    name          = "mesh-ecs-launch-config"
    image_id      = "ami-d61027ad"
    instance_type = "t2.medium"
    iam_instance_profile = "${ecs-instance-role}"
    user_data = "${template_file.user_data.rendered}"
    security_groups = ["${var.security-group-id}"]
    associate_public_ip_address = "true"
    key_name = "kevin-test-pair"
}

resource "template_file" "user_data" {
    template = "${file("${path.module}/user-data.tpl")}"

    vars {
        ecs-cluster-name = "${var.cluster-name}"
    }
}
```
[application-load-balanacer.tf](https://github.com/meshhq/terraform-ecs-cluster/blob/master/ec2/elastic-load-balancer.tf)

```
variable "vpc-id" {}

resource "aws_alb" "mesh-load-balancer" {
    name                = "mesh-load-balancer"
    security_groups     = ["${var.security-group-id}"]
    subnets             = ["${var.subnet-id-1}", "${var.subnet-id-2}"]
}

resource "aws_alb_target_group" "mesh-target_group" {
    name                = "mesh-target-group"
    port                = "80"
    protocol            = "HTTP"
    vpc_id              = "${var.vpc-id}"

    health_check {
        healthy_threshold   = "5"
        unhealthy_threshold = "2"
        interval            = "30"
        matcher             = "200"
        path                = "/"
        port                = "traffic-port"
        protocol            = "HTTP"
        timeout             = "5"
    }
}

resource "aws_alb_listener" "mesh-alb-listener" {
    load_balancer_arn = "${aws_alb.mesh-load-balancer.arn}"
    port              = "80"
    protocol          = "HTTP"

    default_action {
        target_group_arn = "${aws_alb_target_group.mesh-target_group.arn}"
        type             = "forward"
    }
}
```