# Snowflake 3 Stage Data Pipeline Setup with Staging, Transform, Reporting
Quick way to start a 3 stage data pipeline process using Snowflake. This allows the process to broken in to 3 stages (STAGING, TRANSFORM & REPORTING) with different resources, roles & securities that can be assigned to different team members adn/or tools. This script will quickly create Layers in Snowflake using best practices.

It will create Warehouses, roles, users, databases, schemas, usage monitores & security automatically to give you a head start. Below is the diagram of what will be setup and the security for each role.

![](https://github.com/NickAkincilar/Snowflake-3-Stage-Data-Pipeline-Setup-with-Staging-Transform-Reporting/blob/master/images/chart.png?raw=true)






```sql
/* 
--------------------------------------------------------------------------
--------------------------------------------------------------------------
3/24/2020 - Created by Nick Akincilar as a quick start to setup roles & security using best practices.

This script is a quick start to setup a 3 stage (LOAD, TRANSFORM, PROD) data processing layer in Snowflake 
with Virtual Warehouses, Databases, Roles, Schemas, Resource Monitors & Users using best practices 

- WAREHOUSES x 3
  * IMPORT_WH     (MEDIUM, Quick AutoSuspend 15 secs, No Multi-Clustering. INITIALLY_SUSPENDED)
  * TRANSFORM_WH  (MEDIUM, Quick AutoSuspend 15 secs, No Multi-Clustering, INITIALLY_SUSPENDED)
  * REPORTING_WH  (SMALL, AutoSuspend 5 mins to keep it alive for users, Multi-Cluster upto 5, INITIALLY_SUSPENDED)
  

- DATABASES x 2 
  * STAGING
  * PROD

- SCHEMAS x 3
  * STAGING.RAW          (Data Retension 3 days - Used to keep original raw data as it is ingested)
  * STAGING.CLEAN        (Data Retension 3 days -Used to keep cleaned version of raw data to be used for data modeling & ETL)
  * PROD.REPORTING   (Data Retension 90 days - Used by Reporting Users as source for analytics & ETL users as a target after modelling )


  
- ROLES x 3
  *  IMPORT_ROLE    
        ... USAGE only access for Warehouse "IMPORT_WH" (Can't modify/resize)
        ... USAGE on STAGING
        ... USAGE on PROD 
        ... Full access to REPORTING schema in PROD for all existing & new tables
        ... Full access to RAW & CLEAN schemas in STAGING for all existing & new tables
        ... No Access to PROD database
        
  *  TRANSFORM_ROLE (Can read & write to STAGING.Raw + PROD.REPORTING)   
        ... USAGE only access for Warehouse "TRASNFORM_WH" (Can't modify/resize)
        ... Full access to CLEAN schema in STAGING  for all existing & new tables   
        ... Full access to REPORTING schema in PROD for all existing & new tables 
        ... No access to RAW schema in STAGING
 
   *  REPORTING_ROLE (Read-only access to PROD.PROD schema & tables)   
        ... USAGE only access for Warehouse "REPORTING_WH" (Can't modify/resize)
        ... Read-Only access to PROD schema in PROD  for all existing & new tables
        ... No access to STAGING database
  


- USERS x 3  (Test Users)
   * UserReporting  (belongs to REPORTING_ROLE)
   * UserTransform  (belongs to TRANSFORM_ROLE)
   * UserImport     (belongs to IMPORT_ROLE)


- USAGE MONITORS x 3
   * IMPORT_MONITOR     (100 credits a month)
   * TRANSFORM_MONITOR  (100 credits a month)
   * REPORTING_MONITOR  (100 credits a month)

--------------------------------------------------------------------------
--------------------------------------------------------------------------
*/



--------------------------------------------------------------------------
---  MODIFY VARIABLES TO CHANGE ANY RESOURCE NAME
--------------------------------------------------------------------------
set V_ROLE_IMPORT ='IMPORT_ROLE';
set V_ROLE_TRANSFORM ='TRANSFORM_ROLE';
set V_ROLE_BI ='REPORTING_ROLE';

set V_WHNAME_IMPORT ='IMPORT_WH';
set V_WHNAME_TRANSFORM ='ETL_WH';
set V_WHNAME_BI ='REPORTING_WH';

set V_DBNAME_ETL ='STAGING';
set V_DBNAME_PROD ='PROD';

set V_SCHEMA_IMPORT ='RAW';
set V_SCHEMA_CLEAN ='CLEAN';
set V_SCHEMA_PROD ='REPORTING';


set V_MONITOR_TRANSFORM = 'TRANSFORM_MONITOR';
set V_MONITOR_IMPORT = 'IMPORT_MONITOR';
set V_MONITOR_BI = 'REPORTING_MONITOR';

set V_TESTUSER_BI = 'UserReporting';
set V_TESTUSER_TRANSFORM = 'UserTransform';
set V_TESTUSER_IMPORT = 'UserImport';


--------------------------------------------------------------------------
---    1. CREATE IMPORT RESOURCES    
--------------------------------------------------------------------------


-- CREATE A ROLE FOR INGESTING DATA
use role ACCOUNTADMIN;
CREATE ROLE identifier($V_ROLE_IMPORT);
GRANT ROLE identifier($V_ROLE_IMPORT) TO ROLE "SYSADMIN";

--------------------------------------------------------------------------
--- CREATE WAREHOUSE(MEDIUM - NO AUTO CLUSER) FOR INGESTING RAW DATA 
--- SUSPEND IMMEDIATELY (15 secs) AFTER INGESTION JOBS ARE COMPLETE
--------------------------------------------------------------------------

use role SYSADMIN;

CREATE OR REPLACE WAREHOUSE identifier($V_WHNAME_IMPORT) WITH WAREHOUSE_SIZE = 'MEDIUM' WAREHOUSE_TYPE = 'STANDARD' 
INITIALLY_SUSPENDED = TRUE  AUTO_SUSPEND = 15 AUTO_RESUME = TRUE 
MIN_CLUSTER_COUNT = 1 MAX_CLUSTER_COUNT = 1 SCALING_POLICY = 'STANDARD' 
COMMENT = 'Used only for ingesting raw data';

GRANT USAGE ON WAREHOUSE identifier($V_WHNAME_IMPORT) TO Role identifier($V_ROLE_IMPORT);


--------------------------------------------------------------------------
--- CREATE DB, SCHEMA, TABLE, STAGE & CUSTOM_FILEFORMAT  
--- Default Data Retension for Import Schema is 3 days. 
--------------------------------------------------------------------------

use role SYSADMIN;
CREATE DATABASE identifier($V_DBNAME_ETL);

create schema identifier($V_SCHEMA_IMPORT) DATA_RETENTION_TIME_IN_DAYS = 3;

use database identifier($V_DBNAME_ETL);
use schema identifier($V_SCHEMA_IMPORT);


--------------------------------------------------------------------------
--- CREATE SAMPLE TABLE  
--------------------------------------------------------------------------
CREATE TABLE JSON_WEBLOGS ("IMPORTDATE" TIMESTAMP default current_timestamp(), "CONTENT" VARIANT);

--------------------------------------------------------------------------
--- CREATE IMPORT STAGE & SAMPLE JSON FILE FORMAT
--------------------------------------------------------------------------

create or replace stage IMPORT_STAGE
url='azure://something.blob.core.windows.net/somefolder'
credentials=(azure_sas_token='YourSasToken')
file_format = (type = 'CSV');

CREATE FILE FORMAT MYJSON 
TYPE = 'JSON' 
COMPRESSION = 'AUTO' 
ENABLE_OCTAL = FALSE 
ALLOW_DUPLICATE = FALSE 
STRIP_OUTER_ARRAY = TRUE 
STRIP_NULL_VALUES = FALSE 
IGNORE_UTF8_ERRORS = TRUE;

--------------------------------------------------------------------------
---     CREATE CLEAN SCHEMA UNDER STAGING
---     DEFAULT DATA_RETENSION for CLEAN Schema is 3 days
--------------------------------------------------------------------------

create schema identifier($V_SCHEMA_CLEAN) DATA_RETENTION_TIME_IN_DAYS = 3;
use schema identifier($V_SCHEMA_CLEAN) ;

--------------------------------------------------------------------------
--- CREATE SAMPLE TABLE UNDER CLEAN SCHEMA
--------------------------------------------------------------------------
CREATE TABLE WEBLOGS ("IMPORTDATE" TIMESTAMP , C1 STRING, C2 STRING);

--------------------------------------------------------------------------
----        ASSIGN SECURITY ON OBJECTS
--------------------------------------------------------------------------


GRANT USAGE ON WAREHOUSE identifier($V_WHNAME_IMPORT) TO Role identifier($V_ROLE_IMPORT);
grant USAGE On database identifier($V_DBNAME_ETL) to role identifier($V_ROLE_IMPORT);

use schema identifier($V_SCHEMA_IMPORT) ;
grant ALL privileges On schema identifier($V_SCHEMA_IMPORT) to role identifier($V_ROLE_IMPORT);

use schema identifier($V_SCHEMA_CLEAN) ;
grant ALL privileges On schema identifier($V_SCHEMA_CLEAN) to role identifier($V_ROLE_IMPORT);


use schema identifier($V_SCHEMA_IMPORT) ;
grant USAGE On stage IMPORT_STAGE to role identifier($V_ROLE_IMPORT);

use schema identifier($V_SCHEMA_IMPORT) ;
grant USAGE On FILE FORMAT MYJSON  to role identifier($V_ROLE_IMPORT);

--------------------------------------------------------------------------
-- GIVE IMPORT_ROLE FULL ACCESS TO ANY EXITING TABLES 
--------------------------------------------------------------------------

grant ALL privileges on all tables in schema identifier($V_SCHEMA_IMPORT) to role identifier($V_ROLE_IMPORT);
grant ALL privileges on all tables in schema identifier($V_SCHEMA_CLEAN) to role identifier($V_ROLE_IMPORT);
use role SYSADMIN;

--------------------------------------------------------------------------
--- GIVE FULL ACESS TO SECURITYADMIN
--------------------------------------------------------------------------

grant ALL privileges on database identifier($V_DBNAME_ETL) to role SECURITYADMIN;
grant ALL privileges on schema identifier($V_SCHEMA_CLEAN) to role SECURITYADMIN;
grant ALL privileges on schema identifier($V_SCHEMA_IMPORT) to role SECURITYADMIN;
use role securityadmin;
use database identifier($V_DBNAME_ETL);

--------------------------------------------------------------------------
-- GIVE IMPORT_ROLE FULL ACCESS TO ANY FUTURE TABLES 
--------------------------------------------------------------------------

grant ALL privileges on FUTURE tables in schema identifier($V_SCHEMA_IMPORT) to role identifier($V_ROLE_IMPORT);
grant ALL privileges on FUTURE tables in schema identifier($V_SCHEMA_CLEAN) to role identifier($V_ROLE_IMPORT);
use role SYSADMIN;

--------------------------------------------------------------------------
---- CREATE A TEST IMPORT USER 
--------------------------------------------------------------------------

use role SECURITYADMIN;
CREATE OR REPLACE USER identifier($V_TESTUSER_IMPORT) PASSWORD = $V_TESTUSER_IMPORT
LOGIN_NAME = $V_TESTUSER_IMPORT EMAIL = 'userimport@test.com' 
DEFAULT_ROLE = $V_ROLE_IMPORT
DEFAULT_WAREHOUSE = $V_WHNAME_IMPORT MUST_CHANGE_PASSWORD = FALSE;

GRANT ROLE identifier($V_ROLE_IMPORT) TO USER identifier($V_TESTUSER_IMPORT) ;









--------------------------------------------------------------------------
---- 2. CREATE TRANSFORM RESOURCES
--------------------------------------------------------------------------

--------------------------------------------------------------------------
-- CREATE A ROLE FOR TRANSFORMING DATA
--------------------------------------------------------------------------

use role ACCOUNTADMIN;
CREATE ROLE identifier($V_ROLE_TRANSFORM);
GRANT ROLE identifier($V_ROLE_TRANSFORM) TO ROLE "SYSADMIN";

--------------------------------------------------------------------------
--- CREATE WAREHOUSE(SMALL - NO AUTO CLUSER) FOR INGESTING RAW DATA 
--- SUSPEND IMMEDIATELY (5 secs) AFTER INGESTION JOBS ARE COMPLETE
--------------------------------------------------------------------------
use role SYSADMIN;

CREATE OR REPLACE WAREHOUSE identifier($V_WHNAME_TRANSFORM) WITH WAREHOUSE_SIZE = 'MEDIUM' WAREHOUSE_TYPE = 'STANDARD' 
INITIALLY_SUSPENDED = TRUE  AUTO_SUSPEND = 15 AUTO_RESUME = TRUE 
MIN_CLUSTER_COUNT = 1 MAX_CLUSTER_COUNT = 1 SCALING_POLICY = 'STANDARD' 
COMMENT = 'Used only for ETL/Transform';

GRANT USAGE ON WAREHOUSE identifier($V_WHNAME_TRANSFORM) TO Role identifier($V_ROLE_TRANSFORM);




--------------------------------------------------------------------------
--- CREATE PROD DB, SCHEMA, SAMPLE TABLE
--------------------------------------------------------------------------

use role SYSADMIN;
CREATE DATABASE identifier($V_DBNAME_PROD);


--------------------------------------------------------------------------
--- Default Data Retension is set to max 90 days for REPORTING Schema 
--- since this is used for prod reporting 
--------------------------------------------------------------------------
create schema identifier($V_SCHEMA_PROD) DATA_RETENTION_TIME_IN_DAYS = 90;
show schemas;

use DATABASE identifier($V_DBNAME_PROD);
use schema identifier($V_SCHEMA_PROD);
CREATE TABLE WEBLOGS ("IMPORTDATE" TIMESTAMP default current_timestamp(), "S1" STRING);


--------------------------------------------------------------------------
---- ASSIGN READ/WRITE SECURITY ON PROD FOR TRANSFORM ROLE
--------------------------------------------------------------------------

grant USAGE On database identifier($V_DBNAME_PROD) to role identifier($V_ROLE_TRANSFORM);
grant ALL privileges On schema identifier($V_SCHEMA_PROD) to role identifier($V_ROLE_TRANSFORM);

--------------------------------------------------------------------------
-- Full access for Transform Role to PROD schema & existing tables
--------------------------------------------------------------------------
grant ALL privileges on all tables in schema identifier($V_SCHEMA_PROD) to role identifier($V_ROLE_TRANSFORM);
use role SECURITYADMIN;

use role SYSADMIN;
grant ALL privileges on database identifier($V_DBNAME_PROD) to role SECURITYADMIN;
grant ALL privileges on schema identifier($V_SCHEMA_PROD) to role SECURITYADMIN;
use role securityadmin;


use DATABASE identifier($V_DBNAME_PROD);
use schema identifier($V_SCHEMA_PROD);

--------------------------------------------------------------------------
-- Full access for Transform Role to future tables
--------------------------------------------------------------------------
grant ALL privileges on FUTURE tables in schema identifier($V_SCHEMA_PROD) to role identifier($V_ROLE_TRANSFORM);
use role SYSADMIN;



--------------------------------------------------------------------------
-- Assign Usage Only access to Transform Warehouse (No resize or modifications)
-- Assign Usage on ETL database
-- Assign full access to CLEAN Schema under ETL dataabase
-- Assign full access to current & future tables in CLEAN Schema under ETL dataabase
--------------------------------------------------------------------------


GRANT USAGE ON WAREHOUSE identifier($V_WHNAME_TRANSFORM) TO Role identifier($V_ROLE_TRANSFORM);
grant USAGE On database identifier($V_DBNAME_ETL) to role identifier($V_ROLE_TRANSFORM);

use database identifier($V_DBNAME_ETL);
grant ALL privileges On schema identifier($V_SCHEMA_CLEAN) to role identifier($V_ROLE_TRANSFORM);
grant ALL privileges on all tables in schema identifier($V_SCHEMA_CLEAN) to role identifier($V_ROLE_TRANSFORM);
use role SECURITYADMIN;
grant ALL privileges on FUTURE tables in schema identifier($V_SCHEMA_CLEAN) to role identifier($V_ROLE_TRANSFORM);
use role SYSADMIN;

--------------------------------------------------------------------------
---- CREATE A TEST USER FOR TRANSFORM ROLE 
--------------------------------------------------------------------------

use role SECURITYADMIN;

CREATE OR REPLACE USER identifier($V_TESTUSER_TRANSFORM) PASSWORD = $V_TESTUSER_TRANSFORM
LOGIN_NAME = $V_TESTUSER_TRANSFORM EMAIL = 'UserETL@test.com' 
DEFAULT_ROLE = $V_ROLE_TRANSFORM
DEFAULT_WAREHOUSE = $V_WHNAME_TRANSFORM MUST_CHANGE_PASSWORD = FALSE;

GRANT ROLE identifier($V_ROLE_TRANSFORM) TO USER identifier($V_TESTUSER_TRANSFORM);








--------------------------------------------------------------------------
---- 3. CREATE REPORTING RESOURCES
--------------------------------------------------------------------------

--------------------------------------------------------------------------
--- REPORTING ROLES & USERS
--- CREATE A ROLE FOR REPORTING ON PROD DATA
--------------------------------------------------------------------------
use role ACCOUNTADMIN;
CREATE ROLE identifier($V_ROLE_BI);
GRANT ROLE identifier($V_ROLE_BI) TO ROLE "SYSADMIN";



--------------------------------------------------------------------------
--- CREATE WAREHOUSE(SMALL - NO AUTO CLUSER) FOR INGESTING RAW DATA 
--- Multi Cluster up to 5 clusters for Concurrency
--- WAIT TO SUSPEND 5 mins (300 secs) after last query not to loose cache data 
--------------------------------------------------------------------------

use role SYSADMIN;

CREATE OR REPLACE WAREHOUSE identifier($V_WHNAME_BI) WITH WAREHOUSE_SIZE = 'SMALL' WAREHOUSE_TYPE = 'STANDARD' 
INITIALLY_SUSPENDED = TRUE  AUTO_SUSPEND = 300 AUTO_RESUME = TRUE 
MIN_CLUSTER_COUNT = 1 MAX_CLUSTER_COUNT = 5 SCALING_POLICY = 'STANDARD' 
COMMENT = 'Used only for BI Users';

--------------------------------------------------------------------------
--- SET REPORTING WAREHOUSE TIMEOUT AFTER 3 hours (10800 secs) 
--- to stop any run-away queries
--------------------------------------------------------------------------
ALTER WAREHOUSE identifier($V_WHNAME_BI) SET STATEMENT_TIMEOUT_IN_SECONDS=10800;


--------------------------------------------------------------------------
--- Assign Usage only access to REPORTING WAREHOUSE
--------------------------------------------------------------------------
GRANT USAGE ON WAREHOUSE identifier($V_WHNAME_BI) TO Role identifier($V_ROLE_BI);


--------------------------------------------------------------------------
----ASSIGN READ-ONLY SECURITY FOR REPORTING ROLE ON REPORTING DB, WH & SCHEMA
--------------------------------------------------------------------------
use role SYSADMIN;

GRANT USAGE ON WAREHOUSE identifier($V_WHNAME_BI)  TO Role identifier($V_ROLE_BI);
grant USAGE On database identifier($V_DBNAME_PROD) to role identifier($V_ROLE_BI);

use database identifier($V_DBNAME_PROD);
grant USAGE On schema identifier($V_SCHEMA_PROD) to role identifier($V_ROLE_BI);

grant SELECT on all tables in schema identifier($V_SCHEMA_PROD) to role identifier($V_ROLE_BI);
use role SECURITYADMIN;
use database identifier($V_DBNAME_PROD);
grant SELECT on FUTURE tables in schema identifier($V_SCHEMA_PROD) to role identifier($V_ROLE_BI);
use role SYSADMIN;



--------------------------------------------------------------------------
---- CREATE A TEST USER FOR REPORTING 
--------------------------------------------------------------------------

use role SECURITYADMIN;
CREATE OR REPLACE USER identifier($V_TESTUSER_BI) PASSWORD = $V_TESTUSER_BI 
LOGIN_NAME = $V_TESTUSER_BI EMAIL = 'UserBI@test.com' 
DEFAULT_ROLE = $V_ROLE_BI 
DEFAULT_WAREHOUSE = $V_WHNAME_BI MUST_CHANGE_PASSWORD = FALSE;

GRANT ROLE identifier($V_ROLE_BI) TO USER identifier($V_TESTUSER_BI);



--------------------------------------------------------------------------
--- 4. ADD RESOURCE MONITORS WITH 100 CREDITS A MONTH FOR ALERTING
--------------------------------------------------------------------------
USE ROLE ACCOUNTADMIN;
 CREATE RESOURCE MONITOR identifier($V_MONITOR_BI) WITH 
 CREDIT_QUOTA = 100 , frequency = 'MONTHLY', start_timestamp = 'IMMEDIATELY'
 TRIGGERS 
 ON 50 PERCENT DO NOTIFY 
 ON 75 PERCENT DO NOTIFY
 ON 95 PERCENT DO SUSPEND
 ON 100 PERCENT DO SUSPEND_IMMEDIATE ;
 
ALTER WAREHOUSE identifier($V_WHNAME_BI) SET RESOURCE_MONITOR = $V_MONITOR_BI;



   
 CREATE RESOURCE MONITOR identifier($V_MONITOR_IMPORT) WITH 
 CREDIT_QUOTA = 100, frequency = 'MONTHLY', start_timestamp = 'IMMEDIATELY'
 TRIGGERS 
 ON 50 PERCENT DO NOTIFY 
 ON 75 PERCENT DO NOTIFY
 ON 95 PERCENT DO SUSPEND
 ON 100 PERCENT DO SUSPEND_IMMEDIATE ;
 
ALTER WAREHOUSE identifier($V_WHNAME_IMPORT) SET RESOURCE_MONITOR = $V_MONITOR_IMPORT;



 CREATE RESOURCE MONITOR identifier($V_MONITOR_TRANSFORM) WITH 
 CREDIT_QUOTA = 100, frequency = 'MONTHLY', start_timestamp = 'IMMEDIATELY'
 TRIGGERS  
 ON 50 PERCENT DO NOTIFY 
 ON 75 PERCENT DO NOTIFY
 ON 95 PERCENT DO SUSPEND
 ON 100 PERCENT DO SUSPEND_IMMEDIATE ;
 
ALTER WAREHOUSE identifier($V_WHNAME_TRANSFORM) SET RESOURCE_MONITOR = $V_MONITOR_TRANSFORM;

```
