The SAP to Snowflake, ELT, send to salesforce pipeline orchestrates a complete data integration workflow by executing three sequential child pipelines. It handles the end-to-end process of retrieving new order data from SAP, processing and merging that data into Snowflake using ELT operations, and finally sending the processed data to Salesforce.

Snap Summary:

Get new order : Executes the "sap create data" child pipeline to retrieve new order information from SAP systems.
Execute Snowflake SQL : Executes the "elt - merge into snowflake version" child pipeline to perform ELT operations and merge data into Snowflake data warehouse using cloud snaplex.
Send new data to salesforce : Executes the "Excel child" child pipeline to send the processed data to Salesforce for further business operations.

**
The sap create data pipeline retrieves customer information from SAP by executing two different BAPI functions in parallel. The first segment gets detailed information for a specific customer, while the second segment retrieves a list of customers within a specified range. Both results are then merged together to provide comprehensive customer data.

Snap Summary:

Mapper: Maps a hardcoded customer number to the CUSTOMERNO field for detailed customer lookup.
SAP Execute: Executes the BAPI_CUSTOMER_GETDETAIL function to retrieve detailed information for the specified customer, with date conversion format set to yyyyMMdd.
Join: Merges the output streams from both segments into a single combined result using merge join type.
Mapper: Maps customer ID range parameters for the customer list query, setting LOW to 00000001, HIGH to 000010000, and OPTION to BT for the IDRANGE array.
SAP Execute: Executes the BAPI_CUSTOMER_GETLIST function to retrieve a list of customers within the specified range, with date conversion format set to yyyyMMdd.

**
The elt - merge into snowflake version pipeline processes customer order data by joining orders with customer information, transforming and aggregating the data, and then merging the results into a target table based on specific conditions. The pipeline consists of two segments that work together to extract data from source tables, perform transformations and joins, aggregate customer order amounts, and conditionally merge the results into an output table.

Snap Summary:

ELT Select: Selects data from the ORDERS_34R_2KB table in the BIGDATAQA.TEST_DATA schema.
ELT Transform: Transforms the order data by mapping CUST_CODE to CUST_CODE1 and ORD_AMOUNT fields with target expressions for customer and store keys.
ELT Join: Performs an inner join between the transformed order data and customer data using the condition CUST_CODE1 = CUST_CODE.
ELT Transform: Applies additional transformations to the joined data, mapping CUST_CODE and ORD_AMOUNT fields with customer and store key expressions.
ELT Aggregate: Groups the data by CUST_NAME and calculates the sum of order amounts using the SUM function on the _AGR_COLUMN field.
ELT Merge Into: Merges the aggregated data into the MERGE_INTO_OUT_01 table with conditional operations - inserts records when sum > 5000 and not matched, updates when sum < 5000 and matched, and deletes when sum
ELT Select: Selects customer data from the CUSTOMER_50R_3KB table using a custom SQL query that actually queries the ORDERS_34R_2KB table.

**
The email new orders to sales team pipeline processes order data to generate performance reports that are distributed via email and saved as Excel files. It maps order information, then splits the data flow to simultaneously send email notifications to the sales team and create Excel reports with timestamped filenames for record keeping.

Snap Summary:

Map orders : Maps the total_orders_closed field to "Ops Closed 2021" for reporting purposes.
Copy: Splits the data flow into two parallel streams for email notification and file output.
Email Send to sale-team : Sends HTML formatted email notifications to rdill@snaplogic.com with the order performance data.
Excel Formatter: Formats the data into Excel format with a worksheet named "opp_performance".
write order performance file : Writes the Excel formatted data to a timestamped file named "order_performance" with the current date/time and ".xlsx" extension, overwriting any existing file.

*******

The FTP and SAP to SnowFlake pipeline integrates customer data from two sources - a CSV file via FTP and SAP system - then loads the combined data into Snowflake. It reads customer data from an FTP CSV file, retrieves additional customer information from SAP using the GET_CUSTOMERS BAPI, joins the data based on customer number, filters out records with empty first names, maps the fields to target format, and inserts the processed data into a Snowflake table.

Snap Summary:

FTP File Reader : Reads the customer CSV file from the FTP location '../shared/snowflakecustftp.csv'.
CSV Parser: Parses the CSV file using comma delimiter, double quote character, and backslash escape character with auto BOM detection.
Sort: Sorts the CSV data in ascending order by customer number field.
Join: Performs an inner join between the CSV data and SAP data using customer number as the join key.
Filter: Filters out records where the FIRSTNAME field is empty.
Map Customer Data : Maps source fields to target fields including FirstName, LastName, MailingCountry, MailingStreet, MailingCity, MailingPostalCode, and creates OtherState by concatenating first and last names.
Snowflake - Insert: Inserts the processed customer data into the Snowflake table "EBARN"."AE_CUSTOMERS" in the EBARN schema.
SAP Execute: Executes the SAP BAPI '/SAPXCQM/GET_CUSTOMERS' to retrieve customer data with date conversion format 'yyyyMMdd'.
JSON Splitter: Splits the SAP response JSON array at path '$T_CUSTOMER[*]' to create individual customer records.
Sort1 : Sorts the SAP customer data in ascending order by customer number field.

*******

The SAP HANA to SAP BAPI pipeline extracts employee data from an SAP HANA database table, transforms the data by mapping specific fields to a standardized format, and then uses the processed data to create sales orders through an SAP BAPI function. The pipeline connects SAP HANA data retrieval with SAP ERP sales order creation functionality.

Snap Summary:

SAP HANA - Select: Selects employee data from the SAP HANA table "SAP_HANA_DEMO"."sap.hana.democontent.epm.data::MD.Employees" in the SAP_HANA_DEMO schema with autoCommit disabled.
Mapper: Maps employee data fields from the SAP HANA format to standardized field names, transforming EMPLOYEEID to employeeid, NAME.FIRST to firstname, NAME.MIDDLE to middlename, NAME.LAST to lastname, and SEX to sex.
SAP BAPI Order Create : Executes the SAP BAPI function BAPI_SALESORDER_CREATEFROMDAT2 to create sales orders using the mapped employee data, with date conversion format set to yyyyMMdd.

*******

The SFDC-SAP RoundTrip with Reltio Final v4.3 pipeline integrates Salesforce opportunities with SAP sales orders and Reltio entities. It reads opportunity data from Salesforce, creates corresponding sales orders in SAP, updates Salesforce with SAP document numbers, and synchronizes the data with Reltio platform.

Snap Summary:

Opportunity Read : Reads opportunity data including account information and line items from Salesforce using the opportunity ID parameter.
PricebookEntry Read : Retrieves pricebook entry details based on the pricebook entry ID from the opportunity line items.
set order fields : Maps Salesforce opportunity data to SAP order structure with predefined values for document type, sales organization, distribution channel, and other required fields.
SAP Execute: Executes the BAPI_SALESORDER_CREATEFROMDAT2 function to create a sales order in SAP system.
Copy: Splits the data flow to multiple processing paths.
JSON Formatter: Formats data into JSON structure for audit purposes.
Audit Writer BAPI : Writes SAP order creation audit information to a JSON file.
Aggregate: Counts and groups SAP sales documents by opportunity ID and sales document number.
Output Data Mapping : Maps aggregated data to prepare for Salesforce update with opportunity ID and SAP sales document number.
Salesforce Opportunity Update : Updates Salesforce opportunity records with SAP sales document information.
Copy: Splits the updated data flow for further processing.
JSON Formatter: Formats Salesforce update response data into JSON structure.
Audit Writer SFDC Update : Writes Salesforce update audit information to a JSON file.
Salesforce Lookup: Looks up updated opportunity record to retrieve current field values including name, owner, amount, and dates.
Copy: Splits the lookup data for Reltio processing.
JSON Formatter: Formats lookup response data into JSON structure.
Audit Writer SFDC Lookup : Writes Salesforce lookup audit information to a JSON file.
SFDC to Reltio map : Maps Salesforce field names to Reltio-compatible field names for entity creation.
Data : Transforms opportunity created date field for Reltio entity structure.
Document Mapper : Creates Reltio entity JSON structure with opportunity attributes, crosswalks, and relationship information.
Get entity : Wraps the entity data in a postentity structure for API submission.
Create join key : Creates a join key to merge entity data with authentication token.
Join: Performs inner join to combine entity data with Reltio authentication token.
Reltio Post entity : Posts the entity data to Reltio platform using authenticated REST API call.
Format Reltio ID : Extracts and formats the Reltio entity ID from the API response for Salesforce update.
Reltio SFDC Update : Updates Salesforce opportunity with the generated Reltio entity ID.
JSON Formatter: Formats final update response data into JSON structure.
Audit Writer Reltio : Writes Reltio integration audit information to a JSON file.
Get Basic Auth token : Generates authentication credentials and URL for Reltio OAuth token request.
REST Post get auth token : Obtains OAuth access token from Reltio authentication service.
Data : Creates join key for merging authentication token with entity data flow.

*******

The SAP to Oracle pipeline extracts company data from SAP using the BAPI_COMPANY_GETLIST function, processes the retrieved data by splitting JSON objects and mapping fields, and then inserts the transformed data into an Oracle database table. The pipeline handles data transformation including adding metadata fields like update timestamps and user identifiers.

Snap Summary:

Get Companies-SAP : executes the SAP BAPI function BAPI_COMPANY_GETLIST to retrieve company list data with date conversion format set to yyyyMMdd.
Get JSON Object : splits the JSON response by extracting objects from the COMPANY_LIST path to create individual records for each company.
Mapper: transforms the data by mapping COMPANY field to COMPANY_ID, NAME1 field to COMPANY_NAME, and adds metadata fields including UPDATED_BY and LAST_UPDATE .
Oracle - Insert: inserts the transformed company data into the Oracle database table RD_SAP_COMPANY_WITH_DATE_UUID in the DEMO schema.

*******

The SAP HR to Workday Employee pipeline integrates employee data from SAP to Workday by reading employee information from SAP using BAPI calls and then creating or updating personal information in Workday. The pipeline processes employee personnel numbers, retrieves their data from SAP, transforms the data to match Workday's format, and writes the information to Workday while maintaining audit trails and error handling through file outputs.

Snap Summary:

Get PA Data : generates JSON content with employee personnel numbers for employees 0000001005 and 0000001001.
Input For BAPI : maps the PERNR field to EMPLOYEE_ID field to prepare input for SAP BAPI call.
SAP - Read Emp Info : executes the BAPI_EMPLOYEE_GETDATA function in SAP to retrieve employee information with date conversion format yyyyMMdd.
JSON Formatter: formats JSON output from SAP Execute snap.
SAP-ErrFile : writes SAP-related errors to file SS_SAP_Workday_1 with overwrite action.
Format Output : splits the PERSONAL_DATA array from SAP response to process individual records.
MapSAP Fields to WD Attributes : maps SAP fields to Workday Change_Personal_Information structure, including Worker_Reference ID and Gender_Reference mappings.
Create Emp in WD : writes employee personal information to Workday using the Change_Personal_Information object in Human_Resources service.
JSON Formatter: formats JSON output from Workday Write operation.
Write Audit File : creates an audit trail by writing process results to SS_AuditFile_1.json with overwrite action.
JSON Formatter: formats JSON output for error handling from Workday Write.
WD-ErrFile : writes Workday-related errors to file SS_SAP_Workday_2.json with overwrite action.

*******

The SAP to Salesforce pipeline integrates SAP company data with Salesforce by retrieving company information from SAP, transforming the data structure, and creating corresponding Account records in Salesforce. The pipeline uses SAP's BAPI function to fetch company lists and leverages Salesforce's Bulk API for efficient account creation.

Snap Summary:

SAP Execute: executes the BAPI_COMPANY_GETLIST function to retrieve a list of companies from SAP with date conversion format set to yyyyMMdd.
format array : splits the JSON array at the $COMPANY_LIST path into individual company records for processing.
Mapper : maps SAP company fields to Salesforce Account fields, specifically mapping $COMPANY to $SAP_Customer_Number__c and $NAME1 to $Name.
Salesforce Account Create : creates new Account records in Salesforce using the Bulk API with the mapped data from the previous transformation.

*******

The SAP to Redshift pipeline extracts company data from SAP using the BAPI_COMPANY_GETLIST function, processes and transforms the data, and loads it into a Redshift data warehouse. The pipeline retrieves company information, splits the JSON response, maps the fields to target schema, and performs a bulk load operation into Redshift.

Snap Summary:

get companies : executes the SAP BAPI function BAPI_COMPANY_GETLIST to retrieve company data with date conversion format set to yyyyMMdd.
JSON Splitter: splits the JSON response by extracting individual company records from the COMPANY_LIST array.
Mapper: transforms the data by mapping COMPANY to COMPANY_ID, NAME1 to COMPANY_NAME, adds pipeline run UUID as UPDATED_BY, and current timestamp as LAST_UPDATE.
Redshift - Bulk Load: performs bulk load operation to insert the transformed company data into the RD_SAP_COMPANY_WITH_DATE_UUID table in the DEMO schema of Redshift.
