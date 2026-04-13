
Recently, I was testing my Terraform code in OCI to build a three-tier architecture, including network components (VCN, public and private subnets, NAT Gateway, Service Gateway, route tables, and security lists), along with application and database servers.

One of the requirements was to include the OCI region code as a suffix in resource names. For example, if a resource is created for the Chicago region, the naming convention should look like:

```sh
vcn-devops-ord-01
```
(format:resource_name-client-regioncode-sequence)

Environment Context:
---

In my case:
--

- The tenancy home region is Toronto (ca-toronto-1, region key: YYZ)
- The target deployment region is Chicago (us-chicago-1, region key: ORD)

Required Terraform inputs:
---
Below are the key variables required to connect to the OCI tenancy:

```sh
##Necessary fields required for connecting to the target OCI tenancy

user_ocid        = "ocid1.user.oc1..testtesttesttest"
fingerprint      = "3j:3k:test:test"
private_key_path = "/Users/Rajesh.Madhavarao/.ssh/devops_oad.pem"
tenancy_ocid     = "ocid1.tenancy.oc1..testtesttesttest"
compartment_ocid = "ocid1.compartment.oc1..testtesttesttest"
region_name      = "us-chicago-1"
```

Client Identifier:
---

```sh
##Client Name

client_name  = "devops"

```
Fetching region code dynamically

To dynamically fetch the region key (e.g., ORD for Chicago), I used the following data source:

```sh
data "oci_identity_region_subscriptions" "this" {
  tenancy_id = var.tenancy_ocid
  filter {
 name   = "region_name"
    values = [var.region_name]
}
}
```
and exposed via output

```sh
output "target_region_key" {
  value = data.oci_identity_region_subscriptions.this.region_subscriptions[0].region_key
}
```

At this point, everything looked straightforward. I initially implemented the entire setup in a single Terraform layer and attempted to deploy all resources using a single terraform apply. However, this approach led to unexpected issues—especially when dealing with cross-region constraints and identity resources.

This is where the importance of layering Terraform code becomes critical.

What this blog is about
---

In this blog, I’ll walk through a common issue I encountered and explain why it’s important to split Terraform code into layers, instead of managing everything in a single configuration.

Initial approach
---
For testing purposes, I initially built everything in a single Terraform layer, where:

Compartments (Identity)
Networking (VCN, gateways, etc.)
Compute resources

were all defined together.

I then executed a standard workflow:

```sh
terraform plan
terraform apply
```



Now, let us try to execute the Terraform code and apply the resources

```sh

Rajesh.Madhavarao@Eclipsyss-MacBook-Pro regiontest % terraform plan
data.oci_identity_regions.all_regions: Reading...
data.oci_identity_availability_domain.ad: Reading...
data.oci_identity_region_subscriptions.this: Reading...
data.oci_identity_availability_domains.ads: Reading...
data.oci_core_services.all_oci_services[0]: Reading...
data.oci_core_images.all_windows_images: Reading...
data.oci_identity_availability_domain.ad: Read complete after 0s [id=ocid1.availabilitydomain.oc1..aaaaaaaave3zavc6gqeces7x4s4kwoqvyuszk4uiaee3lcma4cq4jb4gmx5q]
.
.
.
.
Plan: 19 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  ~ security_list_id               = "ocid1.securitylist.oc1.us-chicago-1.aaaaaaaae3reg6cpuhd5vokdjagj6ndmibdckhnd4wewi47lf2jei2uusiya" -> (known after apply)
  ~ service_gateway_all_attributes = {
      ~ "0" = {
          ~ block_traffic  = false -> (known after apply)
          ~ compartment_id = "ocid1.compartment.oc1..aaaaaaaatgce3wutmuqmkaatrmly2p2qhqmraqsay2fh4h42c3tftlecstsa" -> (known after apply)
          ~ defined_tags   = {} -> (known after apply)
          ~ id             = "ocid1.servicegateway.oc1.us-chicago-1.aaaaaaaa5zcvjtcu746kpga5szepqtqtdvfjoehp6hqsiwx5ubpmgpod2wrq" -> (known after apply)
          + route_table_id = (known after apply)
          ~ services       = [
              ~ {
                  ~ service_name = "All ORD Services In Oracle Services Network" -> (known after apply)
                    # (1 unchanged attribute hidden)
                },
            ]
          ~ state          = "AVAILABLE" -> (known after apply)
          ~ time_created   = "2026-04-10 16:26:15.053 +0000 UTC" -> (known after apply)
          ~ vcn_id         = "ocid1.vcn.oc1.us-chicago-1.amaaaaaavrxjjqyarv7xhiz52ljfpbspjxe4m7z7cxujoyf7fe2tt4lkrp6a" -> (known after apply)
            # (3 unchanged attributes hidden)
        }
    }
  ~ service_gateway_id             = "ocid1.servicegateway.oc1.us-chicago-1.aaaaaaaa5zcvjtcu746kpga5szepqtqtdvfjoehp6hqsiwx5ubpmgpod2wrq" -> (known after apply)

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply"
now.




Rajesh.Madhavarao@Eclipsyss-MacBook-Pro regiontest % terraform apply
data.oci_core_services.all_oci_services[0]: Reading...
data.oci_identity_availability_domains.ads: Reading...
data.oci_identity_regions.all_regions: Reading...
data.oci_identity_availability_domain.ad: Reading...
data.oci_core_images.all_windows_images: Reading...
data.oci_identity_region_subscriptions.this: Reading...
data.oci_identity_availability_domain.ad: Read complete after 0s [id=ocid1.availabilitydomain.oc1..aaaaaaaave3zavc6gqeces7x4s4kwoqvyuszk4uiaee3lcma4cq4jb4gmx5q]
data.oci_identity_region_subscriptions.this: Read complete after 0s [id=IdentityRegionSubscriptionsDataSource-136365222]
oci_identity_compartment.parent_compartment: Refreshing state... [id=ocid1.compartment.oc1..aaaaaaaa3s3emkreseetotvibn7j4yphcc7cnp6prjp3m46kdza3rcjgjpga]
.
.
.
Plan: 19 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  ~ security_list_id               = "ocid1.securitylist.oc1.us-chicago-1.aaaaaaaae3reg6cpuhd5vokdjagj6ndmibdckhnd4wewi47lf2jei2uusiya" -> (known after apply)
  ~ service_gateway_all_attributes = {
      ~ "0" = {
          ~ block_traffic  = false -> (known after apply)
          ~ compartment_id = "ocid1.compartment.oc1..aaaaaaaatgce3wutmuqmkaatrmly2p2qhqmraqsay2fh4h42c3tftlecstsa" -> (known after apply)
          ~ defined_tags   = {} -> (known after apply)
          ~ id             = "ocid1.servicegateway.oc1.us-chicago-1.aaaaaaaa5zcvjtcu746kpga5szepqtqtdvfjoehp6hqsiwx5ubpmgpod2wrq" -> (known after apply)
          + route_table_id = (known after apply)
          ~ services       = [
              ~ {
                  ~ service_name = "All ORD Services In Oracle Services Network" -> (known after apply)
                    # (1 unchanged attribute hidden)
                },
            ]
          ~ state          = "AVAILABLE" -> (known after apply)
          ~ time_created   = "2026-04-10 16:26:15.053 +0000 UTC" -> (known after apply)
          ~ vcn_id         = "ocid1.vcn.oc1.us-chicago-1.amaaaaaavrxjjqyarv7xhiz52ljfpbspjxe4m7z7cxujoyf7fe2tt4lkrp6a" -> (known after apply)
            # (3 unchanged attributes hidden)
        }
    }
  ~ service_gateway_id             = "ocid1.servicegateway.oc1.us-chicago-1.aaaaaaaa5zcvjtcu746kpga5szepqtqtdvfjoehp6hqsiwx5ubpmgpod2wrq" -> (known after apply)

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

oci_identity_compartment.parent_compartment: Creating...
oci_identity_compartment.parent_compartment: Creation complete after 8s [id=ocid1.compartment.oc1..aaaaaaaaih5xpsjhwkdhupbn3qcqbeaiq5m3ahydu7nr2edcxeuy4h2ovtzq]
oci_identity_compartment.sub_compartments_reco: Creating...
oci_identity_compartment.sub_compartment_sec: Creating...
oci_identity_compartment.sub_compartment_app: Creating...
oci_identity_compartment.sub_compartments_db: Creating...
oci_identity_compartment.sub_compartment_net: Creating...
oci_identity_compartment.sub_compartment_net: Creation complete after 7s [id=ocid1.compartment.oc1..aaaaaaaakrue2gcwdu4sledyykymsgjp3lpbbts2bwto2qkdlejdoxogfbda]
oci_core_vcn.vcn1: Creating...
oci_identity_compartment.sub_compartments_db: Creation complete after 8s [id=ocid1.compartment.oc1..aaaaaaaadauhdsuzihln3dstnx4376bqfbmwzuubjvvevigh5576ndn4o32a]
oci_identity_compartment.sub_compartment_sec: Creation complete after 8s [id=ocid1.compartment.oc1..aaaaaaaaikiwkecjosdkwvqzf5kgsmasoqsftsjuli6j4hqjw7gdibfb4seq]
oci_identity_compartment.sub_compartments_reco: Still creating... [10s elapsed]
oci_identity_compartment.sub_compartment_app: Still creating... [10s elapsed]
oci_identity_compartment.sub_compartments_reco: Creation complete after 11s [id=ocid1.compartment.oc1..aaaaaaaamtcgwcpdcnlrub7lqxcupjfclmfqgy3np57grnvvtbvwt4zxhn4q]
oci_identity_compartment.sub_compartment_app: Creation complete after 14s [id=ocid1.compartment.oc1..aaaaaaaapmzfam3oosgz5grvidcoiwqjilbylskwcnuseryrgsesaufydsza]
╷
│ Error: 404-NotAuthorizedOrNotFound, Authorization failed or requested resource not found.
│ Suggestion: Either the resource has been deleted or service Core Vcn need policy to access this resource. Policy reference: https://docs.oracle.com/en-us/iaas/Content/Identity/Reference/policyreference.htm
│ Documentation: https://registry.terraform.io/providers/oracle/oci/latest/docs/resources/core_vcn 
│ API Reference: https://docs.oracle.com/iaas/api/#/en/iaas/20160918/Vcn/CreateVcn 
│ Request Target: POST https://iaas.us-chicago-1.oraclecloud.com/20160918/vcns 
│ Provider version: 8.6.0, released on 2026-03-17. This provider is 3 Update(s) behind to current. 
│ Service: Core Vcn 
│ Operation Name: CreateVcn 
│ OPC request ID: 5305c5090e9089a8773412f9c841eb26/A618B9CA3783B20A3AF1DC694F5244CE/4755E8BE87500730985423DBD06E68BD 
│ 
│ 
│   with oci_core_vcn.vcn1,
│   on vcn.tf line 1, in resource "oci_core_vcn" "vcn1":
│    1: resource "oci_core_vcn" "vcn1" {

```

As we can see the compartments created successfully but when the terrform code is trying to create dependency resources like VCN we are hitting error like "404-NotAuthorizedOrNotFound, Authorization failed or requested resource not found.", as I have tested this various times and I have sufficient privileges, this issue is not related to IAM policy.

Rather what is happending here is the terraform code is create the compartments, but since compartment is created globally and can be ussed by all regions, it is taking few seconds to be reflecting or bevailable to create network resources>

So how do we handle this scenario,


