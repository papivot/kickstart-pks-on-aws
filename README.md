# Installing PKS on AWS
This document leverages a Ubuntu jumpbox/bastion host to deploy `PKS on AWS`. This is for demo/education purpose only and not intended for production use. Please use Concourse and Platform Automation for deploying in a more formal environment. 

Prerequisites for the installation

* Access to the AWS account. 
*  The following packages installed on the bastion host
	* jq, wget, ruby-full, go-devel
	```console
	sudo apt install jq wget ruby-full go-devel
	```

*  

## Stage 1
### Preparing the jumpbox/bastion
 The following packages need to be installed on the bastion host - 
 * AWS CLI (optional)
	```console
	sudo apt install awscli
	```
* Terraform 
	* Download the **latest** version of Terraform (v0.12.13 in this example) and move to /usr/local/bin
	```console
	wget https://releases.hashicorp.com/terraform/0.12.13/terraform_0.12.13_linux_386.zip
	unzip terraform*.zip
	chmod +x terraform
	sudo mv terraform /usr/local/bin/terraform
	```
* OM Cli
	* Download the **latest** version of OpsMan CLI  - om cli (v4.2.1 in this example) and move to /usr/local/bin
	```console
	wget https://github.com/pivotal-cf/om/releases/download/4.2.1/om-linux-4.2.1
	chmod +x om-linux-4.2.1
	sudo mv om-linux-4.2.1 /usr/local/bin/om
	```
* Pivnet Cli
	* Download the **latest** version of CLI to interact with Pivotal Network  - pivnet cli (v0.0.72 in this example) and move to /usr/local/bin
	```console
	wget https://github.com/pivotal-cf/pivnet-cli/releases/download/v0.0.72/pivnet-linux-amd64-0.0.72
	chmod +x pivnet-linux-amd64-0.0.72
	sudo mv pivnet-linux-amd64-0.0.72 /usr/local/bin/pivnet
	```
* UAA Cli
	* Install the latest Cloudfoundry UAA Cli - uaac 
	```console
	sudo gem install cf-uaac
	```
* Texplate
	* Install the CLI wrapper around Golang text/template - texplate cli and move to  /usr/local/bin
	```console
	mkdir -p $HOME/go/src/github.com/pivotal-cf
	cd $HOME/go/src/github.com/pivotal-cf
	git clone https://github.com/pivotal-cf/texplate.git
	cd texplate/scripts/
	./build
	cd ../out/
	sudo mv texplate_linux_amd64 /usr/local/bin/texplate
	```

## Stage 2
### Setting up the plumbing and installing OpsMan

Download the `paving-pks` git repo. Contact the author, if you do not have access to the repo. 
```console
git clone https://github.com/pivotal/paving-pks.git
cd aws/examples/open-network
```
Within the directory, create a file called `terraform.tfvars` with values specific to your environment - 

```
env_name           = "subdomain"
access_key         = "AAAAAAAAAAAAAAAA"
secret_key         = "sadkjlkasdkasdjkjflkajhfklafHJHJKDHJH"
region             = "us-east-2"
availability_zones = ["us-east-2a", "us-east-2b", "us-east-2c"]
ops_manager_ami    = "ami-06e8270280347fbbe"
rds_instance_count = 0
dns_suffix         = "domainname.com"
vpc_cidr           = "10.0.0.0/16"
```
#### Variables

-   env_name:  **(required)**  An arbitrary unique name for namespacing resources
-   region:  **(required)**  Region you want to deploy your resources to
-   availability_zones:  **(required)**  List of AZs you want to deploy to
-   dns_suffix:  **(required)**  Domain to add environment subdomain to
-   vpc_cidr:  **(default: 10.0.0.0/16)**  Internal CIDR block for the AWS VPC.
-   use_route53:  **(default: true)**  Controls whether or not Route53 DNS resources are created.
-   use_ssh_routes:  **(default: true)**  Enable ssh routing
-   use_tcp_routes:  **(default: true)**  Controls whether or not tcp routing is enabled
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0OTcyMjk5OSwtOTgxNDU1MjAsMTcyNT
YxOTYzOCwtNTY3MjU5Mzc2LC0xNjE3MDgwMTMwLDE2NzA4MzAx
MzgsNjkzMzcxNTA4LDIxMjYwNzA0NjAsLTE2MjI2NDkzNjMsLT
EwMTQ2NjAwMTMsNTkxMDA2NDksLTE3Njc4Mzg0NjRdfQ==
-->