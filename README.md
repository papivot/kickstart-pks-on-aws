# Installing PKS on AWS
This document leverages a Ubuntu jump box/bastion host to deploy `PKS on AWS`. This is for demo/education purpose only and not intended for production use. Please use Concourse and Platform Automation for deploying in a more formal environment. 

Prerequisites for the installation

* Access to the AWS account. 
*  The following packages installed on the bastion host
	* jq, wget, ruby-full, go-devel
	```console
	sudo apt install jq wget ruby-full golang
	```
*  Access to Pivotal Download site - https://network.pivotal.io

## Stage 1
### Preparing the jump box/bastion
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
	pivnet login --api-token=[YOUR_API_TOKEN]
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
**IMPORTANT** Going forward, all jump box activity will be done within this directory. 

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

#### NOTE
For the demo/POC purpose, we will not be deploying/using custom certificates. We will be leveraging self signed certificates. As a result, the following tweaks need to be made to the a file in the git repo downloaded above. 
* Create a new file called `pks-config-new.yml` in the correct directory. 
* Copy and paste the contents from the file in the current Github repo [\[here\]](https://github.com/papivot/kickstart-pks-on-aws/blob/master/pks-config-new.yml)
* Save the file content.
* Update the original `pks-config.yml` file with the new one.
```console
cp -pav ../../../ci/assets/aws/pks-config.yml ../../../ci/assets/aws/pks-config-orig.yml
cp pks-config-new.yml ../../../ci/assets/aws/pks-config.yml
``` 
---

Once the file has been updates, deploy the environment using terraform. 

```console
terraform init
terraform plan -out=plan
terraform apply plan
```

This should deploy the OpsMan VM and all the required AWS artifacts(networking/security groups, DNS etc) necessary to PKS environment. 

The `terraform.tfstate` and `output.tf` file contains the required outputs from the terraform run.

* [IMPORTANT] Modify the DNS configurations, so that the newly created network and DNS domain address is resolvable on the network.  

* The OpsMan UI can be accessed in the browser at the following URL - https://pcf.subdomain.domainname.com.
*  Access the OpsMan UI, accept the certificate warnings and proceed. 
* Select `Internal Authentication` when asked to select the authentication method. 
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

This takes a while (approx. 10+ minutes) and the end result is a fully configured BOSH Director. 

## Stage 4

### Installation of the PKS Control plane.

During this stage, the required product binaries(tiles) and associated stemcell are uploaded to OpsManager. Once done, the tile is configured and the changes applied.

To upload the tile, sign in to https://network.pivotal.io. Once signed in, search for Pivotal Container Service. Select the appropriate version/release - for example - 1.5.1.

Click on the **i** next to the product text. Copy the `pivnet` cli command text and paste it in the bastion shell. For example - 

```console
pivnet download-product-files --product-slug='pivotal-container-service' --release-version='1.5.1' --product-file-id=505925
```
The PKS product file would be named similar to `pivotal-container-service-[version_#]-build.[build_#].pivotal`

Similarly, copy the Linux specific PKS CLI and Kubectl CLI pivnet cli command text and past it in the bastion shell. For example -

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
...
applying errand configuration for the following errands:
	smoke-tests
could not execute "configure-product": configuration not complete.
The properties you provided have been set,
but some required properties or configuration details are still missing.
Visit the Ops Manager for details: pcf.awscloud.navneetv.com
```
This is expected, as we did not provide the api end point certificates (optional) during Stage 1. This can be manually fixed by performing the following steps - 

* Within the bastion host, capture the value of `pks_api_endpoint` from the `tf.output` file
```console
grep pks_api_endpoint tf.output
```
* Login to the OpsManager UI.
* Click on the `Enterprise PKS` tile.  
* Click on `Settings`
* Click on `PKS API` [should be orange in color]
* In the `API Hostname (FQDN) *` field, provide the value captured for `pks_api_endpoint` earlier. 
*  Click `[Generate RSA Certificate]`  and enter the values of `pks_api_endpoint` 
* Click `Generate`
* Click `Save`

Validate that all the remaining setting are completed and `green`. Once validated, you can apply the configuration by either 

* Going back to the main screen of the OpsManager and the clicking on `Review Pending Changes` followed by `Apply Changes`

or

* executing the following command within the Bastion host

```console
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k apply-changes
```
This takes a while (approx. 10+ minutes) and the end result is a functioning PKS API Control plane.

## Stage 5

### Configure PKS Control plane

Once the PKS API VM has been installed and configured, we need to create a user(s) that can access the API with the correct privileges to deploy/manage K8s clusters. 

* Get the UAA Management admin credentials from the PKS tile 
	* This can be done thru the UI or by running the following command - 
```console
om -t [fqdn_opsmanager] -u [opsmansger_admin_user] -p [opsmansger_admin_password] -k credentials -p pivotal-container-service -c .properties.pks_uaa_management_admin_client -t json	
# Copy the value of the secret.
```
* Adjust the VM security group. Make sure that the security group `[pks_api_lb_security_group]` is added to the `pivotal-container-service` EC2 instance in the AWS console. 
* Connect to the PKS UAA
```console
uaac target https://[pks_api_endpoint]:8443 --skip-ssl-validation
```

should return something like this - 
```shell
Target: https://[pks_api_endpoint]:8443
Context: admin, from client admin
```
```console
uaac token client get admin -s [pks_uaa_management_admin_client_credential copied above]
```
should return something like this - 
```shell
Successfully fetched token via client credentials grant.
Target: https://[pks_api_endpoint]:8443
Context: admin, from client admin
```

Create a new user `cody` [example] with the admin privileges.

```console
uaac user add cody --emails cody@example.com -p password
#user account successfully added
uaac member add pks.clusters.admin cody
#success
```

The newly created user `cody` can now leverage the PKS CLI to connect to the API endpoint and manage/create Kubernetes clusters. 

```console
pks login -a [pks_api_endpoint] -u cody -k
```

```shell
Password: ********
API Endpoint: [pks_api_endpoint]
User: cody
Login successful.
```

```console
pks plans
```

```shell
Name   ID                                    Description
small  [PLAN ID]                             Example: This plan will configure a lightweight kubernetes cluster. Not recommended for production workloads.
```

## Destroy

Make sure that all EC2 instances created by the PKS environment, **excluding the OpsMan VM**, has been destroyed.

```console
terraform destroy
```
This will destroy all the plumbing and OpsMan VM, that were created in **Stage 2**.

---
---
# Running Workload on a PKS cluster

## Stage 1 - Deploy a cluster

The first step in deploying a cluster is to login to the pks api and selecting a plan. 
```console
pks login -a [pks_api_endpoint] -u cody -k
```
```shell
Password: ********
API Endpoint: [pks_api_endpoint]
User: cody
Login successful.
```
```console
pks plans
```
```shell
Name   ID                                    Description
small  8A0E21A8-8072-4D80-B365-D1F502085560  Example: This plan will configure a lightweight kubernetes cluster. Not recommended for production workloads.
```

We will use the `small` plan in this example to create a cluster.  To do so execute the following command. The name of the cluster is `cluster00` and the external FQDN for the API server will be  `cluster00.subdomain.domain.com`- 

```console
pks create-cluster cluster00 --external-hostname cluster00.subdomain.domain.com  --plan small
```

```shell
PKS Version:              1.5.1-build.8
Name:                     cluster00
K8s Version:              1.14.6
Plan Name:                small
UUID:                     850bbd0a-1786-4473-86c2-5e46d5e0b9cd
Last Action:              CREATE
Last Action State:        in progress
Last Action Description:  Creating cluster
Kubernetes Master Host:   cluster00.subdomain.domain.com
Kubernetes Master Port:   8443
Worker Nodes:             3
Kubernetes Master IP(s):  In Progress
Network Profile Name:

Use 'pks cluster cluster00' to monitor the state of your cluster
```
It takes around 15-20 mins for the cluster to get deployed. 

---
#### AWS Prerequisites 
Once deployed, we need to perform a quick AWS specific PKS fix on VPC. Copy the UUID of the cluster that is being deployed - `850bbd0a-1786-4473-86c2-5e46d5e0b9cd` in this example. 

Perform the following steps before you create a load balancer:

1.  In the  [AWS Management Console](https://aws.amazon.com/console/), create or locate a public subnet **for each** availability zone (AZ) that you are deploying to. A public subnet has a route table that directs internet-bound traffic to the internet gateway.
     
4.  In the  [AWS Management Console](https://aws.amazon.com/console/), tag **each public subnet** based on the table below, replacing  `CLUSTER-UUID`  with the unique identifier of the cluster. Leave the  **Value**  field empty.
    
    ***Key***
    `kubernetes.io/cluster/service-instance_CLUSTER-UUID`
    
    ***Value***
   ` empty`

---
To validate if the cluster provisioning is complete, run the following command - 

```command
pks cluster cluster00
```
An output containing the following status should indicate a successful completion 

```shell
...
Last Action:              CREATE
Last Action State:        succeeded
Last Action Description:  Instance provisioning completed
...
```
Once the cluster provisioning is complete, adjust the security groups on the master by performing the following steps -

1.  In the  [AWS Management Console](https://aws.amazon.com/console/), filter all the **master** EC2 instances. 
2. For each master instance, `Action` -> `Networking`->`Change security groups`. Select the `pks-master` security group and `Assign Security Groups`

Create a Network Load Balancer to route the traffic to the API servers - 

1. Create a new `Network Load Balancer`, with the following settings - 
	* Load Balancer Port: 8443
	* Select all the AZs and their corresponding public facing subnets within your VPC
	* Create a new `target group`, Target type as 	`Instance`, TCP and port `8443`.
	* Register all the master EC2 instances to the Target group.
2. Update the DNS to advertise the newly created NW LB as an alias to the Kubernetes Master Host FQDN entered in the cluster creation stage. 

Generate the Kubeconfig file for the user using the following command - 

```console
pks get-credentials cluster00
```
where `cluster00` is the name of the cluster that we just created. 

Validate that you can access the cluster by running the following command - 

```console
kubectl get nodes
```
should return  output similar to this - 
```shell
NAME                                      STATUS   ROLES    AGE   VERSION
ip-10-0-10-5.us-east-2.compute.internal   Ready    <none>   57m   v1.14.6
ip-10-0-8-6.us-east-2.compute.internal    Ready    <none>   61m   v1.14.6
ip-10-0-9-5.us-east-2.compute.internal    Ready    <none>   59m   v1.14.6
```

## Stage 2 - Deploy a workload

Download a sample yaml file 

```yaml


```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0MDUzMDc2ODQsLTY1Mjc0OTgwMiwtMT
gwMjIwMzM5LDM0MjU4NzkzLDEyMDEzOTUwNTcsMTYxNDQzNzIz
NywxODMxNTg3NDMwLDg2Nzk3MzQyMCwtMTY2NTEyMTE0LC02Nj
QwNDQyOTEsLTM4NTU4MTkxMSwtOTIzNTY5ODY0LDk1NDczNDk1
MCwzMzA5MjQxOTAsNDgyMTI2MDE4LC04NzMzMTI5NzcsMTYxMD
UzMDIzNCwtMjAxODA3MTkwMiwtMTA4MjU5NTY0MCwtMjAwMTky
MzYxM119
-->