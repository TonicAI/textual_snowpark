# Tonic Textual on Snowpark

This repository provides instructions on how to run Tonic Textual on Snowflake's Snowpark Container Service.  Tonic Textual makes it easy to identify sensitive information within free form text and then either redact or synthesize the information.  It can be used to test ETL pipelines on safe data or for generating synthetic text data for training generative AI models while preventing data leaks.  The benefit of running from Snowpark is that your data all stays within your Snowflake account.  The diagram below shows you how Textual interacts with your Snowflake Data.


![alt text](<snowpark_arch.png>)


# Setup

The following show you how to get started.

## Prerequisites

You must first have access to the Tonic Textual Docker containers.  They are available in Tonic's private Docker repository hosted at [quay.io](https://quay.io).  To gain access you'll need to reach out to us.  The easiest way is to fill out this [form](https://www.tonic.ai/product-demo/textual).  We canm grant you access and provide a limited number of free credits so you can try out our solution on your Snowflake data.

## Install

Installation comes in a few steps.  You'll first need to setup a few things in Snowflake, then push the containers into Snowpark, then stand up the Textual service.

### Initial setup

Let's first create a new database, schema, and role which we'll use to manage the service.

```sql
-- Switch to ACCOUNTADMIN role to create a new role, a new database, and configure a security integration.
USE ROLE ACCOUNTADMIN;

-- Create a role
CREATE ROLE IF NOT EXISTS textual_role;
GRANT ROLE textual_role to USER <User you log in with>;

-- Create a database, grant ownership to role
CREATE DATABASE IF NOT EXISTS textual_db;
USE DATABASE textual_db;
USE SCHEMA public;

CREATE OR REPLACE NETWORK RULE textual_telemetry_egress_access
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('telemetry.tonic.ai');

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION textual_telemetry_egress_access_integration
  ALLOWED_NETWORK_RULES = (textual_telemetry_egress_access)
  ENABLED = true;

GRANT OWNERSHIP ON DATABASE textual_db TO ROLE textual_role;
GRANT USAGE ON INTEGRATION textual_telemetry_egress_access_integration TO ROLE textual_role;
GRANT BIND SERVICE ENDPOINT ON ACCOUNT TO ROLE textual_role;

-- Create a compute pool for the Textual services
CREATE COMPUTE POOL IF NOT EXISTS textual_compute_pool
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = GPU_NV_S;
GRANT USAGE, MONITOR ON COMPUTE POOL textual_compute_pool TO ROLE textual_role;

USE ROLE textual_role;
USE DATABASE textual_db;

-- Create a schema
CREATE SCHEMA IF NOT EXISTS textual_schema;

USE SCHEMA textual_schema;
CREATE IMAGE REPOSITORY IF NOT EXISTS textual_image_repository;
```

### Push containers.

There are 2 docker containers needed to run Textual.  They are:

```
quay.io/tonicai/textual-ml-gpu:latest
quay.io/tonicai/textual-snowflake:latest
```

To pull them down locally, first run ```docker login``` with the credentials we provide. Then pull down both containers:

```
docker pull --platform linux/amd64 quay.io/tonicai/textual-ml-gpu:latest
docker pull --platform linux/amd64 quay.io/tonicai/textual-snowflake:latest
```

We'll now re-tag then before pushing to Snowflake.

```
docker image tag \
  quay.io/tonicai/textual-ml-gpu:latest \
  tonicai-tonic-spcs-test.registry.snowflakecomputing.com/textual_db/textual_schema/textual_image_repository/textual-ml-gpu:latest
docker image tag \
  quay.io/tonicai/textual-snowflake:latest \
  tonicai-tonic-spcs-test.registry.snowflakecomputing.com/textual_db/textual_schema/textual_image_repository/textual-snowflake:latest
```

And finally, we push them to Snowflake.

```
docker push tonicai-tonic-spcs-test.registry.snowflakecomputing.com/textual_db/textual_schema/textual_image_repository/textual-ml-gpu:latest
docker push tonicai-tonic-spcs-test.registry.snowflakecomputing.com/textual_db/textual_schema/textual_image_repository/textual-snowflake:latest
```

You'll repeat the above Docker steps whenever you wish to upgrade.  If you choose a database and schema name that differs from textual_db and textual_schema you'll need to make the corresponding changes in the docker tag and docker push commands as well.

### Stand up services

Now that we've setup the database, schema, role, and pushed the containers to Snowflake we are ready to go live with our services.  Let's create our two services now.


The first service acts as a load balancer for our machine learning models.  You only ever need 1 instances for this service.  It does not require GPU compute and can run a modest CPU.  You will need to provide a SOLAR_SECRET value in the service specification.  It can be any random string and it used to create tokens of your sensitive data.  You can optionally store this SOLAR_SECRET value in a Snowflake SECRET.

```
USE ROLE textual_role;
USE DATABASE textual_db;
USE SCHEMA textual_schema;

-- Create a new instance of the solar service
CREATE SERVICE textual_service
  IN COMPUTE POOL textual_compute_pool
  FROM SPECIFICATION $$
    spec:
      containers:
      - name: textualapi
        image: /textual_db/textual_schema/textual_image_repository/textual-snowflake:latest
        env:
          SOLAR_SECRET: <SECRET>
      endpoints:
        - name: textualendpoint
          port: 9002
          protocol: HTTP
      $$
   MIN_INSTANCES=1
   MAX_INSTANCES=1
   EXTERNAL_ACCESS_INTEGRATIONS=(TEXTUAL_TELEMETRY_EGRESS_ACCESS_INTEGRATION);

```


Now we setup our service which runs the NER models.  We'll start with a single instance but you can scale up the number of instances here to increase throughput.  If you do so, make sure to scale up your COMPUTE POOL resources proportionally.

```
CREATE SERVICE textual_ml
  IN COMPUTE POOL textual_compute_pool
  FROM SPECIFICATION $$
    spec:
      containers:
      - name: textualml
        image: /textual_db/textual_schema/textual_image_repository/textual-ml-gpu:latest
        env:
            TEXTUAL_ML_WORKERS: 4
        volumeMounts:
          - name: textuallogs
            mountPath: /usr/bin/textual/logs_public
        resources:
          requests:
            nvidia.com/gpu: 1
          limits:
            nvidia.com/gpu: 1
      endpoints:
        - name: textualmlendpoint
          port: 7701
          protocol: TCP      
      volumes:
        - name: textuallogs
          source: local
      $$
   MIN_INSTANCES=1
   MAX_INSTANCES=1;
```

### Creating the UDF

You call into the Textual service via a User Defined Function (UDF).  Let's create that UDF now.

```sql
CREATE OR REPLACE FUNCTION textual_redact(x string)
  returns string
  service=textual_db.textual_schema.textual_service
  CONTEXT_HEADERS = (current_user)
  endpoint='textualendpoint'
as '/api/redact';
```

## Install wrap up

With installation complete you can now check on your service.  To grab logs for either service run the following:

```sql
SELECT SYSTEM$GET_SERVICE_LOGS('textual_service', 0, 'textual_api');
SELECT SYSTEM$GET_SERVICE_LOGS('textual_ml', 0, 'textualml');
```

And you can check on the status of the COMPUTE POOL by running:

```sql
SHOW COMPUTE POOLS;
```
# Usage

To begin using Textual on your data just call the UDF in your SQL queries.  For example,

```sql
SELECT textual_redact('Hello, world. My name is Adam and I work at Tonic!')
```

You can also call it on columns of data and have it run a column result-set.  For example, if you have a table of conversational data you could write a query such as:

```sql
SELECT
    conversation,
    textual_redact(conversation) as redacted_convo
FROM
    conversations;
```