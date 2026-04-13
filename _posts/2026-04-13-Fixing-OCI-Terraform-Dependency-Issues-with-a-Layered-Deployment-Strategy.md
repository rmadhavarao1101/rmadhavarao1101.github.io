


Introduction:
---

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


Terraform Execution:
--
The plan is looking perfectly fine

```sh

Rajesh.Madhavarao@Eclipsyss-MacBook-Pro regiontest % terraform plan

Plan: 19 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  ~ security_list_id               = "ocid1.securitylist.oc1.us-chicago-1.test" -> (known after apply)
  ~ service_gateway_all_attributes = {
      ~ "0" = {
          ~ block_traffic  = false -> (known after apply)
          ~ compartment_id = "ocid1.compartment.oc1..test" -> (known after apply)
          ~ defined_tags   = {} -> (known after apply)
          ~ id             = "ocid1.servicegateway.oc1.us-chicago-1.test" -> (known after apply)
          + route_table_id = (known after apply)
          ~ services       = [
              ~ {
                  ~ service_name = "All ORD Services In Oracle Services Network" -> (known after apply)
                    # (1 unchanged attribute hidden)
                },
            ]
          ~ state          = "AVAILABLE" -> (known after apply)
          ~ time_created   = "2026-04-10 16:26:15.053 +0000 UTC" -> (known after apply)
          ~ vcn_id         = "ocid1.vcn.oc1.us-chicago-1.test" -> (known after apply)
            # (3 unchanged attributes hidden)
        }
    }
  ~ service_gateway_id             = "ocid1.servicegateway.oc1.us-chicago-1.test" -> (known after apply)

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply"
now.

```
However, during terraform apply, things started to break.

Terraform successfully created the compartments:

```sh
Rajesh.Madhavarao@Eclipsyss-MacBook-Pro regiontest % terraform apply
 

oci_identity_compartment.parent_compartment: Creating...
oci_identity_compartment.parent_compartment: Creation complete after 8s [id=ocid1.compartment.oc1..test]
oci_identity_compartment.sub_compartments_reco: Creating...
oci_identity_compartment.sub_compartment_sec: Creating...
oci_identity_compartment.sub_compartment_app: Creating...
oci_identity_compartment.sub_compartments_db: Creating...
oci_identity_compartment.sub_compartment_net: Creating...
oci_identity_compartment.sub_compartment_net: Creation complete after 7s [id=ocid1.compartment.oc1..test]
oci_core_vcn.vcn1: Creating...
oci_identity_compartment.sub_compartments_db: Creation complete after 8s [id=ocid1.compartment.oc1..test]
oci_identity_compartment.sub_compartment_sec: Creation complete after 8s [id=ocid1.compartment.oc1..test]
oci_identity_compartment.sub_compartments_reco: Still creating... [10s elapsed]
oci_identity_compartment.sub_compartment_app: Still creating... [10s elapsed]
oci_identity_compartment.sub_compartments_reco: Creation complete after 11s [id=ocid1.compartment.oc1..test]
oci_identity_compartment.sub_compartment_app: Creation complete after 14s [id=ocid1.compartment.oc1..test]
╷
│ Error: 404-NotAuthorizedOrNotFound, Authorization failed or requested resource not found.
│ Suggestion: Either the resource has been deleted or service Core Vcn need policy to access this resource. Policy reference: https://docs.oracle.com/en-us/iaas/Content/Identity/Reference/policyreference.htm
│ Documentation: https://registry.terraform.io/providers/oracle/oci/latest/docs/resources/core_vcn 
│ API Reference: https://docs.oracle.com/iaas/api/#/en/iaas/020292929/Vcn/CreateVcn 
│ Request Target: POST https://iaas.us-chicago-1.oraclecloud.com/20929392929/vcns 
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

As you can see when it attempted to create dependent resources like the VCN, it failed with the above error "Error: 404-NotAuthorizedOrNotFound, Authorization failed or requested resource not found."

It looks like this error is related to IAM policies, but the required permissions are already in place and compartments are already created.

The actual reason is:
---

OCI Identity resources (like compartments) are global, but their propagation across regions is not instantaneous.

In this case:

- Compartments were created in the home region (Toronto / YYZ)
- Terraform immediately tried to use them in the target region (Chicago / ORD)
- Due to propagation delay, the compartments were not yet fully available in the Chicago region

As a result, when Terraform attempted to create the VCN, OCI returned:

```sh
404-NotAuthorizedOrNotFound
```

This is one of the main reasons to split Terraform code into folder-based layers:


```sh

terraform/
├── 01-identity   → compartments
├── 02-network    → VCN, subnets, gateways
├── 03-compute    → app + DB resources

terraform/
│
├── 01-identity/
│   ├── provider.tf
│   ├── variables.tf
│   ├── terraform.tfvars
│   ├── data.tf
│   ├── locals.tf
│   └── compartment.tf
│
├── 02-network/
│   ├── provider.tf
│   ├── variables.tf
│   ├── terraform.tfvars
│   ├── data.tf
│   ├── locals.tf
│   ├── vcn.tf
│   ├── subnet.tf
│   └── output.tf
│
├── 03-compute/
│   ├── application.tf
│   ├── database.tf
│   ├── bastion*.tf
│
└── shared/
    ├── variables.tf (optional common vars)

```

Execution Flow 
---



Instead of a single apply, I executed Terraform in stages:

Step 1: Identity layer

```sh

cd 01-identity
terraform apply \
  -target=oci_identity_compartment.parent_compartment \
  -target=oci_identity_compartment.sub_compartment_app \
  -target=oci_identity_compartment.sub_compartment_net \
  -target=oci_identity_compartment.sub_compartment_sec \
  -target=oci_identity_compartment.sub_compartments_db \
  -target=oci_identity_compartment.sub_compartments_reco

```

Step 2: Network layer

```sh

cd 02-network
terraform apply
```

This ensured compartments were fully propagated before network provisioning began.

Key takeaway
---
Even though Terraform is declarative, cloud APIs are eventually consistent.

So in OCI, especially for Identity → Networking dependencies:

- Compartments must fully propagate before VCN creation
- Splitting Terraform into layers avoids race conditions
- -target can be useful for controlled initialization, but should not be permanent practice

Conclusion
---
This experience reinforced an important lesson:

A single large Terraform stack is simple to write, but not always reliable in real cloud environments like OCI.

Layering Terraform code improves:

- Stability
- Predictability
- Debugging
- Reusability





