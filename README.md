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
	* Download the latest version of Terraform (v0.12.13 in this example) and move to /usr/local/bin
	```console
	wget https://releases.hashicorp.com/terraform/0.12.13/terraform_0.12.13_linux_386.zip
	unzip terraform*.zip
	chmod +x terraform
	sudo mv terraform /usr/local/bin/terraform
	```
* OM Cli
	* Download the latest version of OpsMan CLI  - om cli (v4.2.1 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEzOTI5MTk1NTksNTkxMDA2NDksLTE3Nj
c4Mzg0NjRdfQ==
-->