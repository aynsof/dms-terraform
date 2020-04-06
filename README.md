# DMS Example

## Creates an example AWS DMS for replicating an (on-prem) Oracle database to a cloud-based Postgres database.

## Problem Statement

A problem that a lot of people are finding is that they want to use cloud services, but their data is locked away behind layers of firewalls. As people move more of their applications to the cloud, it’s becoming harder and harder to find app candidates that don’t rely on internal databases.

In many cases it’s an enormous task to migrate the whole database. Aside from the complexity of moving such a large amount of data to the cloud, there are labyrinthine pre-existing pipelines processing and transforming and sending this data to the databases. Updating these pipelines to point to a new cloud-based database would be a colossal undertaking. There is also the question of how unstructured the data is, and how expensive it might be to migrate it without any kind of restructuring.

Application developers don’t want to wait for that to happen. They’re loving the new-found freedom and agility of owning their whole stack, from infrastructure to front-end, and the business owners are loving the massively reduced time-to-production.

There needs to be a way to safely expose the data within these databases to cloud-based applications.

# Database Migration Service

Enter AWS Database Migration Service. Using the DMS Terraform interface, developers can retrieve selected schemas from internal Oracle databases and host them in a read-only Postgres database in the cloud.

This Postgres replica can be hosted in a Virtual Private Cloud (VPC) with absolutely no internet-facing connections except:

 * Highly-available VPN links to an on-premises datacentre/DR site
 * VPC peering connection to an admin VPC
 * VPC peering connections to other applications

The VPC peer connections aren’t transitive, so the risk of exposure to the wider internet is massively reduced.

# DMS Architecture

This is the high-level view of the DMS infrastructure (dms-architecture.png). In this example there are two major on-prem Oracle databases. They are linked by highly-available virtual private network (VPN) connections to the cloud environment. These links provide a secure tunnel through which DMS can communicate with the on-premises databases. There are dual VPN connections from each datacentre to the DMS environment — four connections in total (simplified in the above image to reduce noise).

The DMS infrastructure consists of:
 *  A virtual private cloud (VPC), defining all the networking and security groups.
 *  Two DMS Source Endpoints pointing at each of the Oracle databases. These endpoints define the address, port, credentials, and database type for the source databases.
 *  A DMS Replication Instance, which is used by DMS to house the data while it is being translated from Oracle to Postgres.
 *  A DMS Target Endpoint, which points at a Postgres database that eventually stores and serves the data.
 *  Multiple DMS Replication Tasks, which define the schemas to be migrated. Each of these links together the Source Endpoint, the Replication Instance, and the Target Endpoint.

For applications that want to use the data in the target Postgres database, they need to VPC peer with the DMS VPC. This is a manual process that involves performing a handshake, creating route tables in each VPC, and enabling cross-VPC DNS.

This will create the DMS Endpoints, DMS Replication Instance, and a DMS Task. It’s up to you to point it at a source database — it could be through a VPN to your on-premises infrastructure, or you could create a test RDS instance just to prove that the system works.

# Things to Note

VPN connections aren’t defined in Terraform: a terraform destroy would blow away their Elastic IPs and the on-prem VPN connections would stop working.

Being a managed service, DMS has a fairly high level of opacity. There’s not a lot of insight into its inner mechanics.

# Results

With that said, however, DMS is a solid option for unlocking the data inside organisations’ internal databases. With on-prem environments typically being fairly slow moving and highly restricted, this solution allows developers to reach this data from AWS, enabling new systems to be prototyped and deployed far more rapidly.

## Deployment

### terraform-main
This will create:

 * VPC with a VPN gateway
 * Source and target RDS instances
 * DMS Endpoints for each RDS instance
 * DMS Replication instance

This is designed to be deployed once.

### terraform-repltask
This will create:

 * DMS Replication Task

This is designed to be deployed multiple times.

## Set Variables

Set the following variables in variables.tf:

 *Terraform Remote State*
 * bucket - an s3 bucket that exists in your aws space (see Preperation)
 * lock_table - a dynamodb table in your aws space with the primary key LockID (see Preperation)
 
*Replication main*

Set the following environment variables (substituting the details of your application)

* `export TF_VAR_source_snapshot=<snapshot to restore source database from>`
* `export TF_VAR_source_db_name=<source database name>`
* `export TF_VAR_source_password=<source admin user password>`
* `export TF_VAR_source_username=<source admin user password>`
* `export TF_VAR_source_app_username=<source application user password>`
* `export TF_VAR_source_app_password=<source application user password>`
* `export TF_VAR_target_db_name=<target database name>`
* `export TF_VAR_target_password=<admin user password>`
* `export TF_VAR_target_username=<admin user account name>`

*Replication task*

Edit `_variables.tf`. Update `key = "dblink-repl"` within the s3 backend definition to another value, for example `key = "dblink-appname"`.

Set the following environment variables (substituting the details of your application)
 
 * `export TF_VAR_stack_name=<a unique name for this stack - e.g. dblink-appname>`
 * `export TF_VAR_dms_source_arn=<source endpoint ARN>`
 * `export TF_VAR_dms_target_arn=<target endpoint ARN>`
 * `export TF_VAR_replication_instance_arn=<replication instance ARN>`
 * `export TF_VAR_replication_task_id=<a unique name for this replication task>`

## Creating your infrastructure

1. `terraform init`
2. `terraform plan`
3. `terraform apply`

This command will output your database endpoint, which you will need below.

## Destroying your infrastructure

1. If you have run the infrastructure overnight you will have backups in your backup bucket. You will need to remove these so the terraform can destroy the bucket.
1. `terraform destroy`

This is assuming that you ran `terraform init` previously on the same machine
This command will tear down everything that terraform created.

## VPC Peering

Follow the [VPC Peering guide](https://docs.aws.amazon.com/AmazonVPC/latest/PeeringGuide/vpc-peering-basics.html):

1. Ensure your requester and accepter VPCs do not have overlapping IP address spaces.
1. The owner of the requester VPC sends a request to the owner of the accepter VPC to create the VPC peering connection. 
1. The owner of the accepter VPC accepts the VPC peering connection request to activate the VPC peering connection.
1. Add a route to each VPC's route tables that points to the IP address range of the other VPC.
1. Update security groups to allow traffic from either VPC.
1. Modify your VPC connection to enable DNS hostname resolution.
