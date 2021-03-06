# Cloud Foundry MySQL Service

This project contains a BOSH release of a MySQL service for Cloud Foundry. It utilizes the [v2 broker API](http://docs.cloudfoundry.com/docs/running/architecture/services/writing-service.html).

## MySQL Service Components

<table>
  <tr>
 		<th>Component</th><th>Description</th><th>Build Status</th>
 	</tr>
 	<tr>
 	  <td>CF MySQL Broker</td>
 	  <td>Advertises the MySQL service and plans.  Creates and deletes MySQL databases and
 	  credentials (bindings) at the request of Cloud Foundry's Cloud Controller.
 	  </td>
 	  <td><a href="https://travis-ci.org/cloudfoundry/cf-mysql-broker"><img src="https://travis-ci.org/cloudfoundry/cf-mysql-broker.png" alt="Build Status"></a></td>
 	 </tr>
 	 <tr>
 	   <td>MySQL Server</td>
 	   <td>MySQL 5.6 Server managed by the broker.  Database instances are hosted on this server.
 	   </td>
 	   <td> n/a </td>
 	 </tr>
</table>

## Installation

Prerequisites:

- The CF MySQL service requires a fully working version of [Cloud Foundry (runtime)](https://github.com/cloudfoundry/cf-release).
- Installing the CF MySQL service requires BOSH.
- Instructions on installing BOSH as well as Cloud Foundry (runtime) are located in the [Cloud Foundry documentation](http://docs.cloudfoundry.com/docs/running/deploying-cf/).

Steps:

1. Create a CF MySQL deployment manifest
2. Upload a CF MySQL release to the BOSH director
3. Deploy CF MySQL with BOSH
4. Register the newly created service broker with the Cloud Controller
5. Make the CF MySQL plans public

The MySQL service should now be advertised when running `gcf marketplace`

### Generating a Deployment Manifest

The [generate_deployment_manifest](generate_deployment_manifest) script will help create a deployment manifest.  It requires the following:

- Knowledge about the deployment infrastructure; AWS and vSphere are supported
  - For AWS, you need to know the AZ and subnet id
  - For vSphere, you need you available IP address range, IP address of DNS server
- [Spiff](https://github.com/cloudfoundry-incubator/spiff), installed on the local workstation
- A deployment manifest stub, which is a YAML file with values unique to the deployment (two identical deployments of Cloud Foundry will have stubs with the same keys but some unique values).

To generate your manifest, we recommend the following workflow. 

1. Run the generate_deployment_manifest script. You'll get some error which indicate what the missing manifest parameters are. 
1. Add those paramaters and values into the stub.  See "Hints for Creating a Deployment Manifest Stub" below.
1. Rinse and repeat
1. When all necessary stub parameters are present, the script will output the deployment manifest to stdout. Pipe this output to a file which indicates the environment and the release, such as `cf-mysql-mydevenv.yml`.

####Example:

    $ ./generate_deployment_manifest aws ~/workspace/deployments/deployment-1/stub.yml

    2013/12/16 09:57:18 error generating manifest: unresolved nodes:
	    dynaml.MergeExpr{[jobs mysql properties admin_password]}
	    dynaml.MergeExpr{[jobs cf-mysql-broker properties auth_username]}
	    dynaml.MergeExpr{[jobs cf-mysql-broker properties auth_password]}
	    dynaml.ReferenceExpr{[jobs mysql properties admin_password]}

These errors indicate that the deployment manifest stub is missing the following fields:

    ---
    jobs:
      mysql:
        properties:
          admin_password: <choose_admin_password>
      cf-mysql-broker:
        properties:
          auth_username:
          auth_password:

#### Hints for missing parameters in your deployment manifest stub:

All stubs:

- director_uuid: Shown by running `bosh status`

AWS stubs:

- availability_zone: From the EC2 page of the AWS console, like "us-east-1a".
- subnet_id:  From VPC/Subnets page of AWS console.  Availability zone must match the value set above.

vSphere stubs:

- networks: Follow example from templates/cf-infrastructure-aws.yml.  Set IP addresses.  The networks.subnets.cloud_properties field requires only a sub-field called name.  This should match your vSphere network name, like "VM Network".

Specific to this release:

- admin_password: The admin password for the MySQL server process. You should generate a secure password and configure it using this parameter.
- auth_username: The username cloud controller will use to authenticate with the service broker.
- auth_password: The password cloud controller will use to authenticate with the service broker.

### Upload Release

CF MySQL final releases are stored in the [cf-mysql-release/releases](https://github.com/cloudfoundry/cf-mysql-release/tree/master/releases) directory.  We recommend using the most recent final release.

For example, upload final release 4 by running the following command:

    $ bosh upload release releases/cf-mysql-4.yml

The [cf-release document](http://docs.cloudfoundry.com/docs/running/deploying-cf/common/cf-release.html) provides additional details on uploading releases using BOSH.

### Deploy Using BOSH

Set your deployment using the deployment manifest you generated above.

    $ bosh deployment cf-mysql-mydevenv.yml
    $ bosh deploy

The [Deploying Cloud Foundry with BOSH](http://docs.cloudfoundry.com/docs/running/deploying-cf/vsphere/deploy_cf_vsphere.html) provides additional details on deploying with BOSH.

### Register the CF MySQL Service Broker

1. Login to Cloud Foundry as an admin user
2. Run the following command to register the MySQL broker

    $ gcf create-service-broker cf-mysql BROKER_USERNAME BROKER_PASSWORD URL

    - BROKER_USERNAME and BROKER_PASSWORD are the values you gave for `auth_username` and `auth_password` in the deployment manifest.
    - URL specifies where the Cloud Controller will access the MySQL broker.  If DNS is not configured for the MySQL broker, specify a URL with an IP address such as `http://10.10.34.0`

### Make MySQL Service Plan Public

By default new plans are private, which means they are not visible to end users. This enables an admin to test services before making them available to end users.

To make a plan public, use the old ruby CF CLI (the curl feature will be implemented soon on gcf).


1) Login as an admin user with the ruby cf cli.

    $ cf login

2) Get the service plan guid

    $ cf services -m -t

This returns a JSON response which includes something like the following for each service.

    "service_plans": [
      {
        "metadata": {
          "guid": "a01d462f-a4a4-4945-a008-3ff13c06f719",
          "url": "/v2/service_plans/a01d462f-a4a4-4945-a008-3ff13c06f719",
          "created_at": "2013-11-15T23:42:52+00:00",
          "updated_at": "2013-11-22T18:57:46+00:00"
        },
        "entity": {
          "name": "large",
          "free": false,
          "description": "Large Dummy",
          "service_guid": "e629bb0a-fee7-4a6c-a4f1-9eeec7096c29",
          "extra": "{\"cost\":20.0,\"bullets\":[]}",
          "unique_id": "addonOffering_3365",
          "public": false,
          "service_url": "/v2/services/e629bb0a-fee7-4a6c-a4f1-9eeec7096c29",
          "service_instances_url": "/v2/service_plans/a01d462f-a4a4-4945-a008-3ff13c06f719/service_instances"
        }
      },

3) Set the plan as public, using its GUID.

    $ cf curl PUT /v2/service_plans/a01d462f-a4a4-4945-a008-3ff13c06f719 -b '{"public":'true'}'

4) Verify that the the plan was set to public. Re-run this command and check the 'public' field:

    $ cf services -m -t








