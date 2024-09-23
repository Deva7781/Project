region = "eu-central-1"


# Create the Jenkins instance
resource "aws_instance" "Jenkins"{
  ami = var.linux2_ami
  instance_type = var.micro_instance
  availability_zone = var.availability_zone
  subnet_id = aws_subnet.public_subnet1.id
  key_name = var.key_name
  vpc_security_group_ids = [aws_security_group.jenkins_sg.id]
  user_data = file("jenkins_install.sh")

  tags = {
    Name = "Jenkins"
  }
}

# Create the Ansible instance
resource "aws_instance" "Ansible"{
  ami = var.linux2_ami
  instance_type = var.micro_instance
  availability_zone = var.availability_zone
  subnet_id = aws_subnet.public_subnet1.id
  key_name = var.key_name
  vpc_security_group_ids = [aws_security_group.jenkins_sg.id]
  user_data = file("ansible_install.sh")

  tags = {
    Name = "Ansible"
  }
}

# Create the Sonarqube instance
resource "aws_instance" "Sonarqube"{
  ami = var.ubuntu_ami
  instance_type = var.small_instance
  availability_zone = var.availability_zone
  subnet_id = aws_subnet.public_subnet1.id
  key_name = var.key_name
  vpc_security_group_ids = [aws_security_group.sonarqube_sg.id]

  tags = {
    Name = "Sonarqube"
  }
}


# Create the Grafana instance
resource "aws_instance" "Grafana"{
  ami = var.linux2_ami
  instance_type = var.micro_instance
  availability_zone = var.availability_zone
  subnet_id = aws_subnet.public_subnet1.id
  key_name = var.key_name
  vpc_security_group_ids = [aws_security_group.grafana_sg.id]
  user_data = file("grafana_install.sh")

  tags = {
    Name = "Grafana"
  }
}

# Create the launch configuration for application hosts 
resource "aws_launch_configuration" "app-launch-config" {
  name = "app-launch-config"
  image_id      = "ami-00f22f6155d6d92c5"
  instance_type = "t2.micro"
  security_groups = [aws_security_group.app_sg.id]
  key_name = var.key_name
}

# Create the app autoscaling group
resource "aws_autoscaling_group" "app-asg" {
  name                      = "app-asg"
  max_size                  = 2
  min_size                  = 1
  health_check_type         = "ELB"
  desired_capacity          = 2
  force_delete              = true
  launch_configuration      = aws_launch_configuration.app-launch-config.name
  vpc_zone_identifier       = [aws_subnet.public_subnet1.id, aws_subnet.public_subnet2.id]
  target_group_arns         = [aws_lb_target_group.app-target-group.arn]
}

# Create the app target group
resource "aws_lb_target_group" "app-target-group" {
  name     = "app-target-group"
  port     = 80
  target_type = "instance"
  protocol = "HTTP"
  vpc_id   = aws_vpc.production_vpc.id
}

# Attach the ASG to the target group
resource "aws_autoscaling_attachment" "autoscaling-attachment" {
  autoscaling_group_name = aws_autoscaling_group.app-asg.id
  alb_target_group_arn   = aws_lb_target_group.app-target-group.arn
}

# Create the application Load Balancer
resource "aws_lb" "app-lb" {
  name               = "app-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.app_sg.id]
  subnets            = [aws_subnet.public_subnet1.id, aws_subnet.public_subnet2.id]
}

# Create the listener for the Load Balancer
resource "aws_lb_listener" "app-listener" {
  load_balancer_arn = aws_lb.app-lb.arn
  port              = "80"
  protocol          = "HTTP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app-target-group.arn
  }
}

# Create the S3 bucket for storing Terraform state
resource "aws_s3_bucket" "devops-project-terraform-state" {
  bucket = "devops-project-terraform-state"
  acl    = "private"
  versioning {
    enabled = true
  }

  tags = {
    Name = "Terraform state bucket"
  }
}

# Configure the S3 backend
terraform {
  backend "s3" {
    bucket = "devops-project-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "eu-central-1"
  }
}




output "eip_public_ip" {
  description = "Public IP of the EIP"
  value       = aws_eip.nat_eip.public_ip
}

output "jenkins_public_ip" {
  description = "Public IP of the Jenkins instance"
  value       = aws_instance.Jenkins.public_ip
}

output "ansible_public_ip" {
  description = "Public IP of the Ansible instance"
  value       = aws_instance.Ansible.public_ip
}

output "sonarqube_public_ip" {
  description = "Public IP of the Sonarqube instance"
  value       = aws_instance.Sonarqube.public_ip
}

output "grafana_public_ip" {
  description = "Public IP of the Grafana instance"
  value       = aws_instance.Grafana.public_ip
}

