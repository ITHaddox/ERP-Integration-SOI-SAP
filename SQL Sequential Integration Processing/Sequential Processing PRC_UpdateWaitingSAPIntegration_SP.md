## Overview
`PRC_UpdateWaitingSAPIntegration_SP` is a SQL Server stored procedure designed to manage waiting statuses for SOI (System of Innovation) to SAP integration. It ensures sequential processing for production, adjustments, transfers, and hold status changes in inventory management. The powershell script `SAPIntegrationErrorReport.ps1` was made to simulate the SQL Job that executes the USP. Used for turning on and off the auto execute for debugging manually on dev servers. 

## Purpose
This stored procedure addresses timing conflicts in the SOI to SAP integration process, particularly for the following SAP movement types (MovTypes):
- 501, 502 (Production)
- 201, 202 (Adjustments)
- 303, 304, 305 (Transfers)
- 321, 322, 343, 344, 349, 350 (Hold status changes)

## Key Features
1. **Waiting Status Management**: Sets records to a 'W'aiting status to prevent premature processing in SAP.
2. **Sequential Processing**: Ensures that inventory transactions are processed in the correct order.
3. **Conflict Resolution**: Resolves timing conflicts between different types of inventory movements.
4. **Error Isolation**: Prevents cascading failures by isolating errors to individual records.

## Functionality

### Waiting Records
- Loads all records with 'W'aiting status from PostedManufacturedProductStock (PMPS) and PostedInventoryConsumptions (PIC) tables.
- Default all waiting records to StopWaiting = 1 (True).
- Look for reasons a record needs to continue waiting, and set StopWaiting = 0 (False).

### Pending Transactions
- Identifies pending production and adjustment records not yet in PMPS and PIC tables.
- Handles scenarios where SQL jobs that execute related stored procedures might be delayed.

### Waiting Logic
The procedure determines if a record should continue waiting based on various conditions:

1. **Production**: Wait for all production (501) and reversal (502) to process. Also, check if transaction has happened but not inserted into integration tables yet.
2. **Adjustments**: Wait for positive adjustments (202) and check if they haven't been inserted yet.
3. **Transfers**: Wait for Hold Release records to process before transfer records if product was on hold. (Since one ERP allows transfers while on certain holds statuses and the other doesn't)
4. **Other MovTypes**: Wait for other MovTypes in PMPS or PIC if they were created earlier.
5. **Special K056 Plant Logic**: 
   - 303 MovType waits for 311 to support BMEX's storage location error fixes.
   - 321 or 343 related to K056 transfers wait for 311.

### Stop Waiting
- Updates records to stop waiting (StatusCode = 'N', Resends = 1) if no waiting conditions are met.

### Special Handling for K056 Plant
- Supports WM SAP module changes made to K056 on 9/1/24 for batches already in storage location 9003.


## Supporting USPs and Triggers
There were several other stored procedures and triggers modified that insert records into the PIC and PMPS integration tables.

- Prc_SOIToSAPPostedManufacturedProductStock_SP 
	- Production records do not wait for anything. Reversals wait for production.
- Prc_SOIToSAPPostedInventoryConsumptions_SP  
	- Negative adjustments to quality hold items requires creating a hold release record first due to SAP only allowing adjustments to non-restricted inventory. 
	- Positive adjustments to quality hold items requires creating a hold record after the adjustment.
- TransferDetail_Insert
	- Creates 303 transfer and if on hold then creates hold release record before transfer. 
- TransferDetail_Update
	- Creates 305 receive transfer and if product was on hold then applies hold after receiving. 
- TransferDetail_Delete
	- Creates 304 for reversal of transfer and if product was on hold then apply hold after. 
- InventoryMaintenanceAfterInsert
	- Creates hold and hold release records not done during transfers or adjustments.

## Important Notes
- The procedure uses a cutoff date (@StartDateTime) to limit the scope of records processed.
- A separate cutoff date (@RelatedStartDateTime) is used for related records to prevent indefinite waiting.
- Special logic is implemented for the K056 plant to handle specific scenarios related to storage location 9003.

## Usage
```sql
EXEC [dbo].[PRC_UpdateWaitingSAPIntegration_SP]
```

## Maintenance
- The stored procedure includes logic to handle pre-launch batches for K056 in storage location 9003. This section may need to be removed once all records in the `BatchesPreLaunch_K056_9003` table have `HasShippedOnce = 1`.

## Dependencies
- PostedManufacturedProductStock table
- PostedInventoryConsumptions table
- OK.dbo.ProductionAdjustmentDetail table
- OK.dbo.ProductionAdjustment table
- OK.dbo.Product table
- SOIToSAP.dbo.PlantWarehouse table
- OK.dbo.LotNo table
- MMMovsbyPlant table
- BatchesPreLaunch_K056_9003 table

## Version History
- 2024-06-19: Initial implementation for managing waiting statuses and sequential processing.
- 2024-06-21: Added logic to prevent indefinite waiting for older related records.
- 2024-07-03: Fixed data type issue for LotNoKey in temporary tables.
- 2024-08-05: Added logic for K056 303 wait on 311, removed @StartDateTime condition, changed StopWaiting column to bit datatype.
- 2024-09-10: Added support for pre-launch K056 batches in storage location 9003.

## Author
TMH