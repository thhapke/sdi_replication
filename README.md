# SQL DB to Object Store Replication
(sdi_replication) by thhapke

##Summary 
If there is a requirement to securely replicate a table to an object store this project description and the operators could help you. The original request came from a customer who wants to replicate their ECC/S4 tables to an Azure Data Lake. For the time being that the SLT - SAP DI connection is not general available (planned for Q1 2021) the replication could be done to an HANA DB and from their picked up by the following pipeline. 

## DB to Object Store Mapping Strategy
The low storage cost of an object store is traded for less flexibility, more precisely you cannot as easily as with a database udpate changed data records to csv-files. My proposal is a 2-way approach: Changes are tagged as 'I' for inserts that can right away been appended to an CSV while for updates or deletes tagged as 'U' change-log files are stored and periodically are merged with the basic CSV file. 

## Replication Control Information
In order to securely replicate data, additional control data is needed. This additional information could either be stored to **additional columns** of the data table or as a separate **shadow table** with a foreign key to the data table. The difference is just an additional sql-join. 

The minimum replication controll data:

* **DIREPL_STATUS** Replication Status (Wait, Blocked, Completed)
* **DIREPL_TYPE** Insert, Update, Delete
* **DIREPL_UPDATED** UTC Timestamp of last update
* **DIREPL_PACKAGEID** (optional) Generated by customized SLT script for batch process. This could also be set with DI by an additional preparation step or added to the sql-statement (repl_block-operator) with the *TOP* modfier. 
* **DIREPL_PID** Process ID provided by DI pipeline

## General Process

The general process is basically a sequence of **sql-statement** producer and **SAP HANA Client**-operator finally the **write-File** operation to the **object-store*. The **sql-statement** producer are wrapped into simple custom python operators that most probably will be replaced by standard operators (most likely *structured data operators*) of the next releases. 

1. **Table Dispatcher** gets the tables to be replicated from a table and sends them successively to the output port. It also gets from another input port the messages from processed table packages. It checks if any changes in the table has been detected. If there has been no changes the replication process could stop if the parameter 'Round trips to stop' is set. 
2. **Reset** After a certain define period (latency provided with the table data) the 'blocked' data is again released to be processed in the loop. The reason for not been processed might been a broken pipeline. 
2. **Block** Blocks all records of the lowest 'package Id' waiting to be processed. The blocked records are also updated with an 'pid' (random number provided by DI) 
3. **Select** Selects all blocked records/packageid of this process ('pid')
4. **json-dict** Converts the data of json-format into an csv-format. This could be changed to any other format. Configurable if the replication data should also be stored. 
5. **Write-File** Writes data to the object store. This should be configured either as 'append' or as 'overwrite'. For the later the packageid should be part of the filename to keep the order of the changes. 
5. **Complete** Sets the blocked packageids to 'complete'.

Because the *HANA Client Operator* is not passing the message attributes (It receives only an sql-statement as string at the sql-inport) the attributes has been passed over this operators. The operator 'Merge Attributes' is just doing what the title promises.


## Parallelization
The pipeline is designed to run through a loop of all provided tables and process one packageID. That means one pipeline can cover as many tables as been given throughn the tables table provided to the 'Table Dispatcher'-operator. The replication process could be accelerated by processing multiple table at once. This means that up to the number of tables messages are send through the pipeline shortly one after the other and it is not been waited until a table package has been processed entirely. This results into a 40% performance gain. 