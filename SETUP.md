# Set up the demo environment

## Environment to be set up
- Single VPC
- Single subnet
- Single route table
- Single gateway
- Single Vault server
- Single BIG-IP LTM
- Single NGINX (http) server
- Single NGINX (https) server (optional)
- All-in-one Ansible Automation Platform

The setup looks like the following:
![](./images/acme_managing_cert.png)

## Included contents for set up
### Roles
|Name     |Description|
|:--------|:----------|
|aap      |A role to create an Ansible Automation Platform on AWS EC2.|

### Playbooks
|Name     |Role Used|Description|
|:--------|:--------|:----------|
|`create_networks.yml`|N/A|Create required AWS network resources.|
|`delete_networks.yml`|N/A|Delete AWS network resources created in `create_networks` playbook.|
|`create_vault_vm.yml`|N/A|Create an EC2 instance and set up Vault server.|
|`delete_vault_vm.yml`|N/A|Delete the instance created in `create_vault_vm` playbook.|
|`create_web_http_vm.yml`|N/A|Create an EC2 instance and setup NGINX (http) server.|
|`delete_web_http_vm.yml`|N/A|Delete the instance created in `create_web_http_vm` playbook.|
|`create_web_https_vm.yml`|N/A|Create an EC2 instance and setup NGINX (https) server.|
|`delete_web_https_vm.yml`|N/A|Delete the instance created in `create_web_https_vm` playbook.|
|`create_ltm_vm.yml`|N/A|Create an EC2 instance for BIG-IP LTM.|
|`delete_ltm_vm.yml`|N/A|Delete the instance created in `create_ltm_vm` playbook.|
|`create_aap_vm.yml`|[roles.aap](roles/aap/README.md)|Create an EC2 instance and set up Ansible Automation Controller.|
|`delete_aap_vm.yml`|N/A|Delete the instance created in `create_aap_vm` playbook.|
|`obtain_acme_cert.yml`|N/A|Obtain a new certificate for a specific domain using ACME and store it in the Vault.|
|`create_eip.yml`|N/A|Create an EIP and allocate it to an instance; LTM or NGINX (https) in this case.|
|`config_ltm_https.yml`|N/A|Configure the BIG-IP LTM for using the certificate and forwarding traffic to the NGINX (http). |
|`config_web_https.yml`|N/A|Configure the NGINX (https) for using the certificate.|


## Prerequisites for set up
### Basic requirements for Ansible
Any control node:
- ansible core 2.16+

Ansible collections:
- amazon.aws
- community.crypto
- community.dns
- community.hashi_vault
- f5networks.f5_modules
- redhat.rhel_system_roles


In order for the above collections, you may need to install additional Python libraries.
```
# pip3 install -r requirements.txt
```

### ansible.cfg
Update the following line with your EC2 private key file.
```
[default]
private_key_file = "path to EC2 private key file"
```
### Environment variables
In this setup, you should set the following environment variables on your control node.
```
$ export AWS_DEFAULT_REGION=ap-northeast-1
$ export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
$ export AWS_SECRET_ACCESS_KEY=wJatrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

### Subscriptions
Basically, the demo environment requires access to Red Hat gold image, so you must have a matching Red Hat product subscription and must connect their cloud provider accounts to Red Hat.
Please refer to the [Cloud Access user interface](https://access.redhat.com/management/cloud) or Cloud Sources on [cloud.redhat.com](https://cloud.redhat.com/) as described in the [Red Hat Cloud Access program overview](https://access.redhat.com/documentation/en-us/subscription_central/2023/html/red_hat_cloud_access_reference_guide/getting-started-with-ca_cloud-access).

Required Red Hat subscriptions:
- Red Hat Enterprise Linux for x86_64
- Red Hat Ansible Automation Platform Subscription

Also, the demo uses [F5 BIG-IP Virtual Edition in AWS](https://clouddocs.f5.com/cloud/public/v1/aws_index.html) and [CentOS Stream 9 (x86_64)](https://aws.amazon.com/marketplace/pp/prodview-k66o7o642dfve). So, you need to have each subscription for F5 BIG-IP Virtual Edition and CentOS Stream 9 (basically free) respectively beforehand.

### Domain
For doing this demo, you need to have a domain name. This demo assumes using AWS, so you need to have a public hosted zone in Amazon Route53. Please refer to [Amazon Route53 Document](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingHostedZone.html) for more details.

### ACME capable CA
This demo uses ansible [community.crypto.acme_certficate](https://docs.ansible.com/projects/ansible/11/collections/community/crypto/acme_certificate_module.html#ansible-collections-community-crypto-acme-certificate-module) module and it requires to determine the URL for the ACME CA server API. This demo assumes to use [Let's Encrypt](https://letsencrypt.org/). If you want to know more details about Let's Encrypt, please refer to [Let's Encrypt Docs](https://letsencrypt.org/docs/).

## Usage
### Create required network resources
These variables should be set in group_vars beforehand.
```
aws_vpc: acme_demo_vpc
aws_vpc_cidr_block: 10.2.0.0/16 # adjust with your preference
aws_vpc_subnet_name: acme_demo_subnet
aws_subnet_cidr_block: 10.2.1.0/24 # adjust with your preference
aws_igw_name: acme_demo_gtw
aws_routetable_name: acme_demo_rtb
aws_securitygroup_name: acme_demo_sg

purpose: acme_demo
```
This playbook needs to be run at the beginning.
```
$ ansible-playbook create_networks.yml
```

### Create Vault Server
These variables should be set in group_vars beforehand.
```
aws_vpc_subnet_name: acme_demo_subnet
aws_securitygroup_name: acme_demo_sg
aws_vault_instance_ami: ami-0a7a80a58efb329f5 # ami of CentOS-Stream-9-20250604.1.x86_64-aba856bc-78bf-441c-b25c-980bec33a53f
aws_vault_instance_size: t2.micro

vault_vm_type: vault # should not be modified
vault_vm_name: vault01
purpose: acme_demo
```
And, the following variables are prompted at run-time.
```
aws_keypair_name # Your AWS key pair name corresponding to the private key
```
This playbook can run after running `create_networks` playbook.
```
$ ansible-playbook create_vault_vm.yml
```
Once the playbook successfully finished, you can get the URI for your Vault server. Then you need to login the vm, and make your vault server ready.
```
$ ssh ec2-user@<your vault server public address>

$ export VAULT_ADDR=https://127.0.0.1:8200
$ export VAULT_SKIP_VERIFY=true

$ vault operator init -key-shares=1 -key-threshold=1
```
Vault responds to your initialization request with output containing the unseal key value and initial root token value as shown below.
```
Unseal Key 1: HLOCX7/wwvFg10EMvT9Zfa+5HfiPjKTaJRub06PDMnc=

Initial Root Token: hvs.YOo2eCJlbW3wKIKtj9lHSoz4
```
And, you need to unseal the server:
```
$ vault operator unseal

Unseal Key (will be hidden):
```
NOTE: You need to unseal the server every time the server rebooted. Keep the token, which is used for the subsequent steps.
Please refer to [Vault Docs](https://developer.hashicorp.com/vault/tutorials/get-started/setup) for more details.

### Create Web (http) Server
These variables should be set in group_vars beforehand.
```
aws_vpc_subnet_name: acme_demo_subnet
aws_securitygroup_name: acme_demo_sg
aws_subnet_cidr_block: 10.2.1.0/24 # adjust with your preference
aws_web_instance_ami: ami-0a7a80a58efb329f5 # ami of CentOS-Stream-9-20250604.1.x86_64-aba856bc-78bf-441c-b25c-980bec33a53f
aws_web_instance_size: t2.micro

web_vm_type: web # should not be modified
web_vm_name: web01
purpose: acme_demo

domain_name: "www.yourdomain.net" # adjust with your domain
```
And, the following variables are prompted at run-time.
```
aws_keypair_name # Your AWS key pair name corresponding to the private key
```
This playbook can run after running `create_networks` playbook.
```
$ ansible-playbook create_web_http_vm.yml
```

### Create BIG-IP LTM
These variables should be set in group_vars beforehand.
```
aws_vpc_subnet_name: acme_demo_subnet
aws_securitygroup_name: acme_demo_sg
aws_ltm_instance_ami: ami-0510f3d853d460f2d # F5 BIGIP-21.0.0-0.0.10 PAYG-Good 25Mbps-251104071341-8e1217d4-a046-4cdf-894e-e38175bae37f
aws_ltm_instance_size: m5.xlarge # should not be modified
ltm_vm_type: ltm # should not be modified
ltm_vm_name: ltm01
purpose: acme_demo
```
And, the following variables are prompted at run-time.
```
aws_keypair_name # Your AWS key pair name corresponding to the private key
ltm_passwd # Your password for BIG-IP admin user
```
This playbook can run after running `create_networks` playbook.
```
$ ansible-playbook create_ltm_vm.yml
```

### Create Ansible Automation Controller
These variables should be set in group_vars beforehand.
```
aws_vpc_subnet_name: acme_demo_subnet
aws_securitygroup_name: acme_demo_sg
aws_aap_instance_ami: ami-041b7189cc3449f92 # ami of RHEL-9.6.0_HVM-20250910-x86_64-0-Hourly2-GP3
aws_aap_instance_size: t2.xlarge # should not be modified

aap_vm_type: aap # should not be modified
aap_vm_name: aap01
purpose: acme_demo
```

And, the following variables are prompted at run-time. Also refer to [roles.aap](roles/aap/README.md) for the role details.
```
aws_keypair_name # Your AWS key pair name corresponding to the private key
rhsm_username # Your Red Hat login name
rhsm_passwd # Password for your Red Hat login
aap_admin_passwd # Password for your AAP admin user
aap_pg_passwd # PostgreSQL password for your AAP deployment
```
This playbook can run after running `create_networks` playbook.
```
$ ansible-playbook create_aap_vm.yml
```

### Obtain a certificate
These variables should be set in group_vars beforehand.
```
domain_name: "www.yourdomain.net" # adjust with your domain
country_name: "JP" # adjust with your preference
organization_name: "yourorg" # adjust with your preference
organizational_unit_name: "Dev" # adjust with your preference
acme_email: "admin@{{ domain_name }}" # adjust with your preference
acme_directory: "https://acme-v02.api.letsencrypt.org/directory" # adjust with your preference

files_loc: ./cert-store # adjust with your preference
le_private_key: "{{ files_loc }}/le-account.key" # adjust with your preference
csr_private_key: "{{ files_loc }}/{{ domain_name }}/{{ domain_name }}.key" # adjust with your preference
cert_remaining_days: 30

vault_addr: "https://yourvault.com:8200" # adjust with your vault URI
vault_namespace: "admin"
vault_mount: "cert-store"
vault_cert_path: "{{ domain_name }}"
```
And, the following variables are prompted at run-time.
```
vault_token # Your token for the Vault
```
This playbook can run after running `create_vault_vm` playbook and your vault server is ready (unsealed).
```
$ ansible-playbook create_acme_cert.yml
```
Once you run the playbook, you can find the generated key files and cert files under `{{ files_loc }}` directory. Please keep the `le-account.key` file and encode it base64 for the use of demo.
```
$ tree cert-store
cert-store
├── le-account.key
└── www.yourdomain.net
    ├── www.yourdomain.net-fullchain.crt
    ├── www.yourdomain.net-intermediate.crt
    ├── www.yourdomain.net.crt
    ├── www.yourdomain.net.csr
    └── www.yourdomain.net.key

$ base64 -i le-account.key -o le-account.key.b64
```
### Configure BIG-IP LTM
These variables should be set in group_vars beforehand.
```
vault_addr: "https://yourvault.com:8200" # adjust with your vault URI
vault_mount: "cert-store" # adjust with your preference
vault_cert_path: "{{ domain_name }}"
domain_name: "www.yourdomain.net" # adjust with your domain
```
And, the following variables are prompted at run-time.
```
vault_token # Your token for the Vault
ltm_passwd # Your password for BIG-IP admin user
ltm_pool_host_ip # Private IP address of Web Server (http)
```
This playbook can run after meeting the followings:
- `create_vault_vm` has successfully finished and your vault server is ready (unsealed)
- `create_acme_cert` has successfully finished
- `create_ltm_vm` has successfully finished

```
$ ansible-playbook config_ltm_https.yml
```

### Create Web (https) Server (Optional)
This is an optional step of the demo, where you can deploy an NGINX as HTTPS server instead of the combination of BIG-IP and NGINX HTTP server. These variables should be set in group_vars beforehand.
```
aws_vpc_subnet_name: acme_demo_subnet
aws_securitygroup_name: acme_demo_sg
aws_subnet_cidr_block: 10.2.1.0/24 # adjust with your preference
aws_web_instance_ami: ami-0a7a80a58efb329f5 # ami of CentOS-Stream-9-20250604.1.x86_64-aba856bc-78bf-441c-b25c-980bec33a53f
aws_web_instance_size: t2.micro

web_vm_type: web # should not be modified
web_vm_name: web02
purpose: acme_demo

domain_name: "www.yourdomain.net" # adjust with your domain
```
And, the following variables are prompted at run-time.
```
aws_keypair_name # Your AWS key pair name corresponding to the private key
```
This playbook can run after running `create_networks` playbook.
```
$ ansible-playbook create_web_https_vm.yml
```

### Configure Web (https) Server (Optional)
This step is the pair of the above creating web (https) server. Please note this and configuring BIG-IP LTM are mutually exclusive. These variables should be set in group_vars beforehand.
```
vault_addr: "https://yourvault.com:8200" # adjust with your vault URI
vault_mount: "cert-store" # adjust with your preference
vault_cert_path: "{{ domain_name }}"
domain_name: "www.yourdomain.net" # adjust with your domain
```
And, the `hosts` variable in the playbook needs to be aligned with `web_vm_name` at the previous step.
```
hosts: tag_Name_web02
```

Also, the following variables are prompted at run-time.
```
vault_token # Your token for the Vault
```
This playbook can run after meeting the followings:
- `create_vault_vm` has successfully finished and your vault server is ready (unsealed)
- `create_acme_cert` has successfully finished
- `create_web_https_vm` has successfully finished

```
$ ansible-playbook config_web_https.yml
```

### Create an EIP
These variables should be set in group_vars beforehand. The variable `eip_instance_name` should be the same as `ltm_vm_name` or `web_vm_name` (optionally).
```
aws_vpc_subnet_name: acme_demo_subnet
aws_securitygroup_name: acme_demo_sg
aws_eip_name: acme_demo

purpose: acme_demo
eip_instance_name: "{{ ltm_vm_name }}"
```
This playbook need to be run at the last of set up.
```
$ ansible-playbook create_eip.yml
```

### Clean up the environment
All the delete resource playbooks corresponding to each create resource playbook are avaialble. Those playbooks can run assuming related variables have already set previously.
