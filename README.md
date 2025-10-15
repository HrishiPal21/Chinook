# Chinook Data Warehouse Project

This is my data warehouse project for DAMG7370. I built an ETL pipeline that moves data from an Azure SQL database into Snowflake using Azure Data Factory. The goal was to transform transactional data into a proper analytical data warehouse.

## What This Project Does

The project takes data from the Chinook database (a sample music store database) and loads it into a star schema data warehouse in Snowflake. Instead of just copying tables, I transformed the structure to make it optimized for analytics and reporting.

The main components are:
- Azure Data Factory pipeline that extracts data from Azure SQL
- Staging area in Snowflake where raw data lands
- Final data warehouse with dimension and fact tables
- MERGE statements that handle incremental loads

## Technology Stack

- Azure Data Factory for orchestration
- Azure SQL Database as the source system
- Snowflake as the data warehouse
- Azure Key Vault for credential management

## The Data Model

I designed a star schema with one fact table and four dimension tables.

The fact table is SALES_FACT which contains 412 invoice transactions. It has foreign keys to the dimension tables and measures like total sale amount.

The dimension tables are:
- DATE_DIM with 13,149 dates covering 2000 to 2035
- TIME_DIM with 1,440 rows (every minute of the day)
- CUSTOMER_DIM with 59 customer records
- ARTIST_DIM with 275 artists

The important design choice here was using DATE_DIM_KEY in the fact table instead of storing actual dates. This means I can easily filter and aggregate by year, quarter, month, or weekday without complex date functions.

## Key Implementation Details

One of the main requirements was implementing hash-based change detection. Instead of comparing every column to detect if a customer changed, I calculate an MD5 hash of all their attributes. When loading data, I only update records where the hash value is different. This is much more efficient than comparing 10+ columns individually.

The ADF pipeline uses a ForEach loop with parameters. I pass it an array like ["chinook.Invoice", "chinook.InvoiceLine"] and it processes each table. This makes the pipeline reusable - I can add more tables just by updating the parameter array.

All dimension tables use surrogate keys (CUSTOMER_KEY, ARTIST_KEY, etc.) that are independent from the source system. These are generated using Snowflake sequences and make the warehouse more maintainable long-term.

## What's in the Repository

The sql folder contains all the Snowflake scripts including table creation, data population, and MERGE statements.

The adf folder has the exported pipeline JSON files from Azure Data Factory.

The docs folder has the project documentation with screenshots showing the pipeline execution and data validation.

## Data Volumes

The pipeline loaded:
- 412 invoices from the source system
- 2,240 invoice line items
- 59 customers
- 275 artists

The dimension tables were populated with:
- 13,149 dates in DATE_DIM
- 1,440 time slots in TIME_DIM

## Validation and Testing

I ran several validation queries to confirm everything worked correctly.

First, I checked that all tables were populated with the expected row counts.

Second, I verified that every customer record has a hash value - this proved the hash calculation logic is working.

Third, and most importantly, I ran a query that joins SALES_FACT to DATE_DIM using DATE_DIM_KEY. This returned results showing invoice details along with derived date attributes like month name and weekday. This proves the dimensional model is working correctly.

Here's an example of what that query returns:
- Invoice 1, Date Key 20090101, Full Date 2009-01-01, Month Jan 2009, Weekday Thursday
- Invoice 2, Date Key 20090102, Full Date 2009-01-02, Month Jan 2009, Weekday Friday

## Challenges and Solutions

The biggest challenge was getting the ADF pipeline to work with Snowflake. I kept getting 404 errors about missing parquet files in blob storage. After troubleshooting, I realized the issue was with how the pipeline was staging data. I switched from Table mode to Query mode in the Copy activity and that resolved it.

Another issue was schema name duplication. The pipeline was looking for "chinook.chinook.Invoice" because it was adding the schema prefix twice. I fixed this by using a split expression to extract just the table name from the parameter.

I also ran into case sensitivity issues with Snowflake. The tables were created in uppercase but the pipeline was sending mixed case names. Adding an upper() function to the table name parameter fixed this.

Finally, my DATE_DIM and TIME_DIM tables were initially empty because I created them but forgot to run the INSERT statements. Once I added the population scripts using Snowflake's GENERATOR function, both tables loaded successfully.

## What I Learned

This project really helped me understand the difference between transactional and analytical database design. In OLTP systems you normalize everything, but in a data warehouse you denormalize into star schemas for query performance.

I also learned why surrogate keys are important. They make the warehouse independent from the source system and make it easier to handle slowly changing dimensions in the future.

The hash-based change detection was a new concept for me. It's way more efficient than comparing every column, especially when you have wide tables with many attributes.

On the technical side, I got a lot better at debugging ADF pipelines and understanding how data flows between Azure services and Snowflake.

## Running This Project

If you want to replicate this setup:

Start by running the SQL scripts in the sql folder to create all the tables and sequences in Snowflake.

Then set up your Azure Data Factory with linked services pointing to your Azure SQL database and Snowflake account.

Import the pipeline JSON from the adf folder and update the connection parameters.

Run the pipeline to load the staging tables, then execute the MERGE scripts to load the final dimension and fact tables.

The SQL scripts have comments explaining what each section does.

## Project Requirements

This project met all the assignment requirements:
- Created star schema with DATE_DIM, TIME_DIM, dimension tables, and fact table
- Built functional ADF pipeline with ForEach loop
- Implemented MD5 hash-based change detection in CUSTOMER_DIM
- Used DATE_DIM_KEY foreign key instead of raw dates in SALES_FACT
- Used MERGE statements for upsert operations
- Validated all data loads with test queries
- Documented everything with screenshots

## About

This was completed as part of DAMG7370 Database Management and Database Design at Northeastern University, Fall 2025.

The project demonstrates ETL pipeline development, dimensional modeling, and cloud data warehouse implementation using modern tools.
