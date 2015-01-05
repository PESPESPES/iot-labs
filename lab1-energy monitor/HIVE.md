#Batch Processing with Hive/Sqoop#


The following details how to **manually** run a Hive job to process the energy readings that have been written to blob storage.  This agregates the data and calculates averages per hour per device.  The result will be written to a MSSQL database using Sqoop.

While this is currently a manual process, it can be automated using Oozie

###Pre-requisites###
- Setup hadoop cluster in Azure.  Ensure the Storage Account is the same as the one prepared for the **worker role** prepared in the previous lab **The one that pulls from the Event Hub to blob storage**
- Enable remote access to the cluster
- Download json serde **(shall we just provide this as-is?)**

http://repo1.maven.org/maven2/com/proofpoint/hive/hive-serde/1.0/hive-serde-1.0.jar

- Create new directory `jars` in the storage account created when setting up the cluster
- Upload the jar to that location
- Create a MS SQL instance in Azure, copy the JDBC connection string **(for simplicity avoid special characters in the password, ! character for instance causes problems when running the Sqoop job**)

###Create database table###

- Connect to the MSSQL database and execute the following script to create a new database.

```sql
	CREATE TABLE averagesPerHour (
    deviceId]  VARCHAR (255) NOT NULL,
    hourOfDay] VARCHAR (2)   NOT NULL,
    average]   INT           NOT NULL,
    PRIMARY KEY (deviceId, hourOfDay))
```

###Running the Hive Job###

Apache Hive provides a means of running MapReduce job through an SQL-like scripting language, called HiveQL. Hive is a data warehouse system for Hadoop, which enables data summarization, querying, and analysis of large volumes of data.

- Navigate to https://<yourclustername>.azurehdinsight.net
- Navigate to the Hive Editor tab
- Execute the following script 

```sql
	ADD JAR wasb:///jars/json-serde-1.1.9.3-SNAPSHOT.jar;
	set hive.exec.dynamic.partition = true;
	set hive.exec.dynamic.partition.mode = nonstrict;
	
	 CREATE EXTERNAL TABLE energyreadings (
	    timestamp string, deviceId string, startReading int, endReading int, energyUsage int
	  )
	  --PARTITIONED BY (year string, month string, day string)
	  ROW FORMAT 
	    serde 'org.openx.data.jsonserde.JsonSerDe'
	  STORED AS TEXTFILE
	  LOCATION '/2014/12/11';
	
	create external table averagesByHour (hour string, average string) 
	row format delimited 
	fields terminated by '\t' 
	lines terminated by '\n' 
	stored as textfile location 'wasb:///output';
	  
	insert into table  averagesByHour select deviceId, hour(from_unixtime(unix_timestamp(timestamp, "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"))), avg(energyUsage) from energyreadings where deviceId is not NULL group by hour(from_unixtime(unix_timestamp(timestamp, "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"))), deviceId ;

```

- Monitor the job and once complete navigate the storage account and examine the file in the output folder.  This would be expected to contain values which we can write to a database.

###Sqoop###

Sqoop is a tool designed to transfer data between Hadoop clusters and relational databases. You can use it to import data from a relational database management system (RDBMS) such as SQL or MySQL or Oracle into the Hadoop Distributed File System (HDFS), transform the data in Hadoop with MapReduce or Hive, and then export the data back into an RDBMS. 

- Connect to the HDInsight cluster via RDP and start the Hadoop command line
- Navigate to C:\apps\dist\sqoop-1.4.4.2.1.9.0-2196\bin
- Execute the following command (modify as appropriate)

	`sqoop export --connect jdbc:sqlserver://<server>.database.windows.net:1433;database=testTedIotDb;user=<userame>@<server>;password=<password>;encrypt=true;hostNameInCertificate=*.database.windows.net;loginTimeout=30; --table averagesPerHour`

- Once complete data should be available in the database