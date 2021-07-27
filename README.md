# EzCSR

This module simplifies the creation of a CSR1KV in AWS with with External Connections to Aviatrix Gateways for simulating on-prem environments. The resources created by the module are pictured below:

![Topology](aviatrix-demo-onprem-Topology.png)

# Instructions

This Module requires the AWS Provider and Aviatrix Providers to be defined and configured in the root module. The Module accepts Aviatrix Transit Gateway data sources as inputs and automatically creates the external connections on the Aviatrix Controller and configures the CSR with the correct crypto settings and Tunnels. Module invocation looks like so:

```terraform
# Optionally, create multiple aliased providers to deploy multiple CSRs across regions
provider "aws" {
	alias   = "use1"
	region  = "us-east-1"
}

provider "aws" {
	alias   = "usw2"
	region  = "us-west-2"
}

# Call the Module and provide necessary info
# public connections (using public EIP) and private connections (using private IP over DX or VPC peering) are defined separately to allow mix-and-match
# format for the connection definition is: 
# "<Aviatrix Transit Gateway Name>:<Aviatrix Transit Gateway BGP AS Number>:<Number of tunnels ***This currently only supports 1***>"
# NOTE: Number of tunnels parameter is reserved for future use to support multi-vrf use-cases, but currently can only be set to 1.
module "demo-onprem-1" {
  source                                        = "github.com/gleyfer/aviatrix-demo-onprem"
  providers                                     = { aws = aws.use1 }
  hostname                                      = "CSROnprem-East1"
  vpc_cidr                                      = "172.16.0.0/16"
  public_sub                            = "172.16.0.0/24"
  private_sub                           = "172.16.1.0/24"
  instance_type                         = "t2.medium"
  public_conns                          = [ "Test-Transit:64525:1", "TestWest-Transit:64526:1"]
  private_conns                         = [ "Test-Transit:64525:1" ]
  csr_bgp_as_num                        = "64527"
  create_client                         = true
}

module "demo-onprem-2" {
  source                                        = "github.com/gleyfer/aviatrix-demo-onprem"
  providers                                     = { aws = aws.usw2 }
  hostname                                      = "CSROnprem-West2"
  vpc_cidr                                      = "172.16.0.0/16"
  public_sub                            = "172.16.0.0/24"
  private_sub                           = "172.16.1.0/24"
  instance_type                         = "t2.medium"
  public_conns                          = [ "Test-Transit:64525:1", "TestWest-Transit:64526:1"]
  private_conns                         = [ "Test-Transit:64525:1" ]
  csr_bgp_as_num                        = "64528"
  create_client                         = true
}
```

Example of getting outputs:

```terraform
# Get Public EIP of CSR
output "CSREast1_Pub_IP" {
  value = module.demo-onprem-1.public_ip
}

# Get command to directly SSH into the CSR via EIP
# Ex: ssh -i private_key.pem ec2-user@<CSR EIP>
output "CSREast1_SSH_Cmd" {
    value = module.demo-onprem-1.ssh_cmd_csr
}

# Get command to ssh into test client via forwarded port
# Ex: ssh -i private_key.pem ec2-user@<CSR EIP> -p 2222
output "CSREast1_SSH_Cmd_Client" {
    value = module.demo-onprem-1.ssh_cmd_client
}

# Get generated CSR configuration/user_data for debugging
output "CSREast1_User_Data" {
    value = module.demo-onprem-1.user_data
}
```

Explanation of module arguments:

- **hostname:** The hostname which will be configured on the CSR and which will prefix all of the resources created.
- **key_name:** If you have an existing SSH key in AWS which you would like to use for the CSR & test client login you can put the name here. CSR will automatically be provisioned an admin user with a password of: Password123!
- **vpc_cidr:** The VPC CIDR block to use when creating the VPC which the CSR will reside in.
- **public_sub:** The public subnet for the CSR public facing interface.
- **private_sub:** The private subnet for the CSR private facing interface. If enabled, the test client will be created in this subnet.
- **instance_type (optional):** The instance type to launch the CSR with. Default is t2.medium
- **public_conns:** List of public external connection definitions (please see above example for format). Tunnels will be created to primary and hagw automatically.
- **private_conns:** List of private external connection definitions (For DX, please see above example for format). Tunnels will be created to primary and hagw automatically.
- **create_client (optional):** If enabled, creates an amazon linux instance in the private subnet, configures public SG to allow port 2222 and configures a port forward on the CSR to allow SSH into the test instance using the CSR EIP and port 2222. SSH Key used for the instance will either be the one specified in key_name or generated automatically. Disabled by default.

Module outputs:

- **public_ip:** Public EIP of CSR instance
- **hostname:** Hostname of CSR Instance
- **ssh_cmd_csr:** SSH Command to directly access the CSR
- **ssh_cmd_client:** SSH Command to access the test client