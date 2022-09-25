# streaming-from-qldb-to-opensearch

The goal of this tutorial is to understand and implement a solution that streams data changes made on a ledger on Amazon QLDB to an OpenSearch Domain.
A common use case for this architecture would be a scenario where is required an event-driven architecture to process data and perform analytics in near real-time for data stored on a ledger on QLDB.


1. [Architecture](#architecture)
2. [Considerations](#considerations)
3. [Deploying the infrastructure using CloudFormation](#deploying-the-infrastructure-using-cloudformation)
4. [Configuring Access Control in OpenSearch](#configuring-access-control-in-opensearch)
5. [Inserting data in the Ledger](#inserting-data-in-the-ledger)
6. [Querying the data in OpenSearch](#querying-the-data-in-opensearch)
7. [Conclusion](#conclusion)


## Architecture

![Architecture-Diagram](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/architecture-diagram.png "Architecture-Diagram")

The way it works is as follows:
    1) There is a Stream associated with the QLDB ledger. Every change made on the ledger is captured in the stream which sends that data to a Kinesis Stream (note that the QLDB stream is near-real time)
    2) Once the data is in Kinesis Stream, consumers can process it. In this case to move the data to the destination (OpenSearch domain) a Kinesis Firehose Delivery Stream is configured. The delivery stream has associated a data transformation lambda function to filter only the relevant events we want to insert into OpenSearch.


## Considerations
- The configuration settings are in a basic mode and most of them has the default value. For a production workload these configurations should be changed to select an optimal value. For example: 
- The OpenSearch domain is created in a Public Networking mode but for a production scenario the VPC mode offers more control and security.
- The Kinesis Firehose delivery stream buffers are configured at the lowest value possible to see the data being moved to its destination as soon as possible, but for a production scenario these values should be carefully chosen.
- OpenSearch Domain Master user and password are configured as paramaters in the CFN template. However, there are more secure alternatives such as using Parameter Store parameters.


## Deploying the infrastructure using CloudFormation
We will create all the components showed in the architecture diagram using a CloudFormation template. After that, some manual configurations will be required.
The template can be found in [this GitHub repository](https://github.com/omarrosadio/streaming-from-qldb-to-opensearch) with the name *streaming-from-qldb-to-opensearch.yaml*.

The parameter values you would need to change are:
    - OpenSearchMasterUsername: The master username you will use to connect to the OpenSearch service
    - OpenSearchMasterPassword: The master password you will use to connect to the OpenSearch service. Passwords must contain at least one uppercase letter, one lowercase letter, one number, and one special character.
    
Create the Stack and before continue we need all components created. This can take up to 30 minutes (most of the time is consumed in the OpenSearch domain creation).

![Cloudformation-Stack](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/cloudformation-stack.jpg "Cloudformation-Stack")


## Configuring Access Control in OpenSearch

The IAM Role created that will be used by Kinesis Firehose to move the data to the OpenSearch Domain has the required IAM Permissions to do that (actions that includes *es*). However, there is required to explicitly allow and assign the permissions from the OpenSearch Domain. To do this we need to configure these permissions entering to the dashboard.

The dashboard URL can be obtained from the OpenSearch console or from the CloudFormation Stack - Outputs section with the name "OpenSearchDashboard":

![Cloudformation-Outputs](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/cloudformation-outputs.jpg "Cloudformation-Outputs")


Use the username and password entered as input parameters during Stack creation:

![Opensearch-Login](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/opensearch-login.jpg "Opensearch-Login")

Then go to the hamburger menu in the top left corner -> "Security" -> "Roles":

![Opensearch-Roles](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/opensearch-roles.jpg "Opensearch-Roles")

For this demo, select the "all_access" role and in "Mapped users" section click on "Manage mapping":

![Opensearch-Manage-Mapping](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/opensearch-manage-mapping.jpg "Opensearch-Manage-Mapping")


And add the Kinesis Firehose Delivery Stream Role ARN in the "Backend roles" section. This value can be found in CloudFormation Stack - Outputs section with the name "KinesisFirehoseRole".

![Opensearch-Backend-Roles](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/opensearch-backend-roles.jpg "Opensearch-Backend-Roles")

And click on Map button.


## Inserting data in the Ledger

Now that everything is configured, we need to insert some sample data to see the solution in action.

Go to "Amazon QLDB" -> "PartiQL editor" on the web console. Select the recently created ledger (qldb-stream) and run the following script to create a table, an index and insert some data into the table.

    CREATE TABLE Person;
    CREATE INDEX ON Person (GovId);
    INSERT INTO Person VALUE {'GovId':'AAA-BB-0001', 'GovIdType': 'Passport', 'FirstName':'Raul', 'LastName':'Lewis', 'Country': 'USA', 'Age': 23}; 
    INSERT INTO Person VALUE {'GovId':'CCC-DD-0002', 'GovIdType': 'Passport', 'FirstName':'Brent', 'LastName':'Logan', 'Country': 'USA', 'Age': 40};
    INSERT INTO Person VALUE {'GovId':'EEE-FF-0003', 'GovIdType': 'Passport', 'FirstName':'Alexis', 'LastName':'Pena', 'Country': 'USA', 'Age': 29};
    INSERT INTO Person VALUE {'GovId':'GGG-HH-0004', 'GovIdType': 'Passport', 'FirstName':'Melvin', 'LastName':'Parker', 'Country': 'Mexico', 'Age': 18};
    INSERT INTO Person VALUE {'GovId':'III-JJ-0005', 'GovIdType': 'DNI', 'FirstName':'Salvatore', 'LastName':'Spencer', 'Country': 'Mexico', 'Age': 21};
    INSERT INTO Person VALUE {'GovId':'LLL-MM-0006', 'GovIdType': 'DNI', 'FirstName':'Carlos', 'LastName':'Trump', 'Country': 'Peru', 'Age': 42};
    INSERT INTO Person VALUE {'GovId':'NNN-OO-0007', 'GovIdType': 'DNI', 'FirstName':'John', 'LastName':'Connor', 'Country': 'Chile', 'Age': 36};
    INSERT INTO Person VALUE {'GovId':'PPP-QQ-0008', 'GovIdType': 'Other', 'FirstName':'Diana', 'LastName':'Brown', 'Country': 'Brazil', 'Age': 30};
    INSERT INTO Person VALUE {'GovId':'RRR-SS-0009', 'GovIdType': 'Other', 'FirstName':'Albert', 'LastName':'Johnson', 'Country': 'Peru', 'Age': 29};
    INSERT INTO Person VALUE {'GovId':'TTT-UU-0010', 'GovIdType': 'Other', 'FirstName':'Freddy', 'LastName':'Rose', 'Country': 'Ecuador', 'Age': 33};


The statements should be executed successfully:
![QLDB-insert-data](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/qldb-insert-data.jpg "QLDB-insert-data")


## Querying the data in OpenSearch

After the data is inserted in the ledger, it can take up to 1 minute to be send from QLDB Stream to Kinesis Data Stream. And then up to 1 minute from Kinesis Data Firehose to OpenSearch. So after about 2 minutes from data insertion it should be visible on OpenSearch.

To validate the index was successfully created go to the hamburger menu in the top left corner -> "Query Workbench" and execute the query:

    SHOW tables LIKE %;

And an index with the name we use as an input parameter in the CFN template should appear:

![Opensearch-Index-Validation](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/opensearch-index-validation.jpg "Opensearch-Index-Validation")

If the index is not listed, then validate step 4 and make sure the role ARN was granted with the corresponding permissions.

To see the data on OpenSearch go to the hamburger menu in the top left corner -> "Visualize" and click on "Create index pattern"

![Opensearch-Index-Pattern-1](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/opensearch-index-pattern-1.jpg "Opensearch-Index-Pattern-1")

On "Index pattern name" field enter the index name configured during Stack creation (*OpenSearchIndexName* parameter) and click on "Next step":

![Opensearch-Index-Pattern-2](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/opensearch-index-pattern-2.jpg "Opensearch-Index-Pattern-2")

And finally click on "Create index pattern":

![Opensearch-Index-Pattern-3](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/opensearch-index-pattern-3.jpg "Opensearch-Index-Pattern-3")


The inserted data on the ledger is visible on OpenSearch and new changes will also be streamed to OpenSearch. Go to the hamburger menu -> "Discover" section to see it:

![Opensearch-Index-Data](https://raw.githubusercontent.com/omarrosadio/streaming-from-qldb-to-opensearch/master/images/opensearch-index-data.jpg "Opensearch-Index-Data")

## Conclusion

Having the ability to analyze data in real or near real-time is crucial for analytics workloads. QLDB's supports this using Streams which sends the data to a Kinesis Stream. Once the data is on Kinesis Stream it can be processed in multiple ways. In this example it is delivered to a OpenSearh destination.
I hope this example could help and could be used as a starting point for custom implementations. With the base infrastructure more components can be added: more delivery streams consuming from Kinesis Data Stream to delivery in parallel to another targets e.g. S3, Redshift, New Relic, etc.
