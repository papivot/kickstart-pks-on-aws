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

Download the `paving-pks` git repo. Contact the author if you do not have access to the repo. 
```console
git clone https://github.com/pivotal/paving-pks.git
cd aws/examples/open-network
```
Within the directory, create a variables file called `terraform.tfvars` with values specific to your environment - 

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
-   rds_instance_count: **(default:0)**
-   ops_manager_ami: **(optional)** Ops Manager AMI, get the right AMI according to your region from the AWS guide downloaded from [Pivotal Network](https://network.pivotal.io/products/ops-manager) (if set to `""` no Ops Manager VM will be created)
-  rds_instance_count: **(default: 0)** Whether or not you would like an RDS for your deployment

Once the file has been updates, deploy the environment using terraform. 

```console
terraform init
terraform plan -out=plan
terraform apply plan
```

This should deploy the Opsman VM and all the required AWS artifacts(networking/security groups, DNS etc) necessary to PKS environment. 

The `terraform.tfstate` and `output.tf` file contains the required outputs from the terraform run.

* [IMPORTANT] Modify the DNS configurations, so that the newly created network and DNS domain address is resolvable on the network.  

* The OpsMan UI can be accessed in the browser at the following URL - https://pcf.subdomain.domainname.com.
*  Access the OpsMan UI, accept the certificate warnings and proceed. 
* Select `Internal Authentication` when asked to select the autoentication method. 
* Enter the values for `Username`, `Password`, `Password confirmation`, `Decryption Password`, `Decryption Password confirmation`
* If using Proxy, enter the proxy server specific configurations. 
* Accept the EULA and press `Setup Authentication`

## Stage 3
### BOSH Director configuration and installation

Leveraging the `terraform.tfstate` file, the BOSH director is configured. While inside the directory where the Stage2 was executed, run the following commands - 

```console
jq -e -r '.outputs|map_values(.value)' terraform.tfstate > tf.output
texplate execute ../../../ci/assets/aws/director-config.yml -f tf.output -o yaml > director-config.yml
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k configure-director --config director-config.yml
```

this should produce an output similar to this - 

```shell
started configuring director options for bosh tile
finished configuring director options for bosh tile
started configuring availability zone options for bosh tile
...
finished configuring network assignment options for bosh tile
started configuring resource options for bosh tile
finished configuring resource options for bosh tile
```

Upon successful configuration, the following changes need to be applies to create and configure the BOSH Director VM. This can be done by -

```console
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k apply-changes
```

This takes a while (approx 10+ minutes) and the end result is a fully configured BOSH Director. 

## Stage 4

### Installation of the PKS Control plane.

During this stage, the required product binaries(tiles) and associated stemcell are uploaded to OpsManager. Once done, the tile is configured and the changes applied.

To upload the tile, sign in to https://network.pivotal.io. Once signed in, search for Pivotal Container Service. Select the appropriate version/release - for example - 1.5.1.

Click on the **i** next to the product text. Copy the `pivnet` cli command text and paste it in the bastion shell. For example - 

```console
pivnet download-product-files --product-slug='pivotal-container-service' --release-version='1.5.1' --product-file-id=505925
```
The PKS product file would be named similar to `pivotal-container-service-[version_#]-build.[build_#].pivotal`

Similarly, copy the Linux specific PKS CLI and Kubectl CLI picnet cli command text and past it in the bastion shell. For example -

```console
pivnet download-product-files --product-slug='pivotal-container-service' --release-version='1.5.1' --product-file-id=496006 # For Linux specific PKS CLI
pivnet download-product-files --product-slug='pivotal-container-service' --release-version='1.5.1' --product-file-id=484672 # For Linux specific Kubectl CLI
```

Find the link to the relevant stemcell in the Product Description tab and download the **IaaS specific** stemcell.  For e.g.

```console
pivnet download-product-files --product-slug='stemcells-ubuntu-xenial' --release-version='315.81' --product-file-id=460228
```
The stemcell file would be named similar to `light-bosh-stemcell-[version_#]-aws-xen-hvm-ubuntu-xenial-go_agent.tgz`

Once these 4 files have been downloaded, the PKS Product file and the stemcell needs to be uploaded to the OpsManager,

This is done by the following commands - 

```console
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k upload-product  -p [name_of_the_product_file]
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k upload-stemcell -s [name_of_stemcell_file]
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k stage-product   -p pivotal-container-service -v [version_# e.g. 1.5.1-build.8]
```

Similar to the BOSH Director, the PKS tile needs to be configured. The can be done similar to the BOSH Director by running the following commands - 

```console
texplate execute ../../../ci/assets/aws/pks-config.yml -f tf.output > pks-config.yml
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k configure-product --config pks-config.yml
```

This should partially configure the tile with the output similar to this - 

```shell
configuring pivotal-container-service...
setting up network
finished setting up network
setting properties
...
a1
{"errors":{".pivotal-container-service.pks_tls":["Cert pem can't be blank","Cert pem is invalid","Private key pem can't be blank","Private key pem is invalid"]}}
0
```

This is e
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTMwMzg2NTgzMiwtMjAwMTkyMzYxMywtMT
kwNjIzMzA5NSwxNTEyMjc1NDMwLC00NTY1OTEwNzQsNTQ3NTEx
MTExLDE0MjI2MzY4NDgsMzQ5MDYyOTM2LC0xNzkxMDYxNTczLC
04OTgwMjEyNTEsLTEzMjA4MTQ0OSwtOTgxNDU1MjAsMTcyNTYx
OTYzOCwtNTY3MjU5Mzc2LC0xNjE3MDgwMTMwLDE2NzA4MzAxMz
gsNjkzMzcxNTA4LDIxMjYwNzA0NjAsLTE2MjI2NDkzNjMsLTEw
MTQ2NjAwMTNdfQ==
-->