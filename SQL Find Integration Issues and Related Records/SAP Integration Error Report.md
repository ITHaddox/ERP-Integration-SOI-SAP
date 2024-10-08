## Overview
`RPT_SAPIntegrationErrorReport_SP` is a SQL Server stored procedure designed to monitor and analyze the processing of inventory transactions in an SAP integration environment. It focuses on finding failed or waiting records and their related history, providing a comprehensive view of batch lifecycles.

## Purpose
This stored procedure addresses the need to:
1. Identify failed ('X') or long-waiting ('W') records in the integration process.
2. Retrieve the complete history of related records for each affected batch.
3. Display the most recent error message for each problematic record.
4. Group and order records by batch for easy analysis.

## Key Features
1. **Error and Waiting Record Identification**: Finds records with 'X' status or 'W' status that have been waiting longer than a specified threshold.
2. **Related Record Retrieval**: Gathers all related records for each identified batch, providing a complete transaction history.
3. **Latest Error Message Retrieval**: Uses partitioning to fetch the most recent error message for each record, even if the status has changed.
4. **Comprehensive Data Collection**: Pulls data from multiple tables including PostedInventoryConsumptions (PIC), PostedManufacturedProductStock (PMPS), and MMMovsByPlant (MBP).
5. **MovType Description Mapping**: Includes a description for each movement type for better readability.
6. **Pre/Post Release Differentiation**: Flags records as occurring before or after a specific release datetime.

## Functionality

### Input Parameters
- `@StartSearchDateTime`: Start of the date range to search for problematic records.
- `@EndSearchDateTime`: End of the date range to search for problematic records.

### Process Flow
1. Creates temporary tables for MovType descriptions and various record types.
2. Populates temporary tables with data from PIC, PMPS, and MBP tables, focusing on failed or long-waiting records.
3. Retrieves related records for each identified batch.
4. Combines all records and orders them by batch and creation date.

### Output
The procedure returns a result set containing detailed information about each record, including:
- Record identifiers (RecordKey, TableN)
- Status information (StatusCode, MovType, MovTypeDescription)
- Quantity and UOM
- Batch and Plant details
- Dates (CreateDateTime, InterfaceDate)
- Error messages
- Various other transaction-specific fields

## Usage
```sql
EXEC [dbo].[RPT_SAPIntegrationErrorReport_SP] 
    @StartSearchDateTime = '2000-08-01 00:00:00', 
    @EndSearchDateTime = '2024-09-06 00:00:00'
```

## Excel Visualization
The output of this stored procedure is typically exported to an Excel spreadsheet for further analysis and visualization. The spreadsheet includes conditional formatting to highlight groups of records for each batch, making it easier to track the lifecycle of inventory transactions.

![[IMG-20241008093614591.png]]

Key aspects of the Excel visualization:
- Each row represents a transaction related to a specific batch.
- Columns include details such as RecordKey, TableN, StatusCode, MovType, MovTypeDescription, Quantity, UOM, Batch, Plant, StorageLoc, ReceivingPlant, CreateDateTime, InterfaceDate, and SAPMessage.
- Color coding is used to differentiate between different status codes or other significant attributes.
- Batches are grouped together, allowing for easy tracking of a product's journey through various transactions.

## Maintenance
- The `@BeforeReleaseDateTime` variable is set to a specific date and time (2024-06-21 12:02:00). This may need to be updated for future releases or analyses.
- The `@WaitingTooLong` variable is set to 60 minutes. Adjust this value if the threshold for "waiting too long" changes.

## Dependencies
- PostedInventoryConsumptions table
- PostedManufacturedProductStock table
- MMMovsByPlant table
- Corresponding audit tables for each of the above

## Version
Find SAP Errors and Related Record History V6.2 USP Version

## Author
TMH - Tyler Matthew Haddox