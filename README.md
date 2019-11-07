# Installing PKS on AWS
This document leverages a Ubuntu jumpbox/bastion host to deploy `PKS on AWS`. This is for demo/education purpose only and not intended for production use. Please use Concourse and Platform Automation for deploying in a more formal environment. 

Prerequisites for the installation

* Access to the AWS account. 
*  
*  

## Stage 1
### Preparing the jumpbox/bastion
 The following packages need to be installed on the bastion host - 
 * AWS CLI (optional)
	```console
	sudo apt install awscli
	```
* JQ
	```console
	sudo apt install jq
	```
* Terraform 
	* Download the latest version of Terraform (v0.12.13 in this example)
	```console
	wget https://releases.hashicorp.com/terraform/0.12.13/terraform_0.12.13_linux_386.zip
	
	
	```
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTAyOTI0MjEzMiwtMTc2NzgzODQ2NF19
-->