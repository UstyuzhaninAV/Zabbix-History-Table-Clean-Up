# Zabbix-History-Table-Clean-Up
Zabbix History Table Clean Up

Zabbix history table gets really big, and if you are in a situation where you want to clean it up.
Then we can do so, using the below steps.
Stop zabbix server.
Take table backup - just in case.
Create a temporary table.
Update the temporary table with data required, upto a specific date using epoch.
Move old table to a different table name.
Move updated (new temporary) table to original table which needs to be cleaned-up.
Drop the old table. (Optional)
Restart Zabbix
Since this is not offical procedure, but it has worked for me so use it at your own risk.
### Step 1 
#### Stop the Zabbix server

**Comand**

	sudo service zabbix-server stop

**Script.**

	echo "------------------------------------------"
	echo "    1. Stopping Zabbix Server            "
	echo "------------------------------------------"
	sudo service zabbix-server stop;

### Step 2 
#### Table Table Backup.
**Comand**

	mysqldump -uzabbix -pzabbix zabbix history_uint > /tmp/history_uint.dql

**Script.**

	echo "------------------------------------------"
	echo "    2. Backing up ${ZABBIX_TABLE_NAME} Table.    "
	echo "    Location : ${BACKUP_FILE_PATH}        "
	echo "------------------------------------------"
	mkdir -p ${BACKUP_DIR_PATH}
	mysqldump -u$ZABBIX_USER -p$ZABBIX_PASSWD $ZABBIX_DATABASE ${ZABBIX_TABLE_NAME} > ${BACKUP_FILE_PATH}

### Step 3 
#### Open your favourite MySQL client and create a new table
**Comand**
	CREATE TABLE history_uint_new_20161007 LIKE history_uint;


**Script.**

	echo "------------------------------------------------------------------"
	echo "    3. Create Temp (${ZABBIX_TABLE_NAME}_${EPOCH_NOW}) Table"
	echo "------------------------------------------------------------------"
	echo "CREATE TABLE ${ZABBIX_TABLE_NAME}_${EPOCH_NOW} LIKE ${ZABBIX_TABLE_NAME}; " | mysql -u$ZABBIX_USER -p$ZABBIX_PASSWD $ZABBIX_DATABASE;
	
### Step 4
#### Insert the latest records from the history_uint table to the history_uint_new table

**Getting epoch time in bash is simple.**
*Current Date.*

	date --date "20160707" +%s

*Date 3 Months Ago.*

	date --date "20161007" +%s

Here is the output.

	[ahmed@localhost ~]$ date --date "20160707" +%s
	1467829800
	[ahmed@localhost ~]$ date --date "20161007" +%s
	1475778600

Now insert data for 3 months.

	INSERT INTO history_uint_new SELECT * FROM history_uint WHERE clock > '1413763200';
	Script.
	echo "------------------------------------------------------------------"
	echo "    4. Inserting from ${ZABBIX_TABLE_NAME} Table to Temp (${ZABBIX_TABLE_NAME}_${EPOCH_NOW}) Table"
	echo "------------------------------------------------------------------"
	echo "INSERT INTO ${ZABBIX_TABLE_NAME}_${EPOCH_NOW} SELECT * FROM ${ZABBIX_TABLE_NAME} WHERE clock > '${EPOCH_3MONTHS_BACK}'; " | mysql -u$ZABBIX_USER -p$ZABBIX_PASSWD $ZABBIX_DATABASE;
### Step 5 
#### Move history_uint to history_uint_old table
**Comand**

	ALTER TABLE history_uint RENAME history_uint_old;

**Script.**

	echo "------------------------------------------------------------------"
	echo "    5. Rename Table ${ZABBIX_TABLE_NAME} to ${ZABBIX_TABLE_NAME}_${EPOCH_NOW}_old"
	echo "------------------------------------------------------------------"
	echo "ALTER TABLE ${ZABBIX_TABLE_NAME} RENAME ${ZABBIX_TABLE_NAME}_${EPOCH_NOW}_old;" | mysql -u$ZABBIX_USER -p$ZABBIX_PASSWD $ZABBIX_DATABASE;

### Step 6 
#### Move newly created history_uint_new to history_uint
**Comand**

	ALTER TABLE history_uint_new_20161007 RENAME history_uint;

**Script.**

	echo "------------------------------------------"
	echo "    6. Rename Temp Table (${ZABBIX_TABLE_NAME}_${EPOCH_NOW}) to Original Table (${ZABBIX_TABLE_NAME})"
	echo "------------------------------------------"
	echo "ALTER TABLE ${ZABBIX_TABLE_NAME}_${EPOCH_NOW} RENAME ${ZABBIX_TABLE_NAME}; " | mysql -u$ZABBIX_USER -p$ZABBIX_PASSWD $ZABBIX_DATABASE;
### Step 7
#### [OPTIONAL] Remove Old Table.

As we have backed-up the table we no long need it. So we can drop the old table.
**Comand**

	DROP TABLE hostory_uint_old;

**Script.**

	echo "------------------------------------------"
	echo "    7. Dropping Old Table (${ZABBIX_TABLE_NAME}_${EPOCH_NOW}_old), As we have already Backed it up. "
	echo "------------------------------------------"
	echo "DROP TABLE ${ZABBIX_TABLE_NAME}_${EPOCH_NOW}_old; " | mysql -u$ZABBIX_USER -p$ZABBIX_PASSWD $ZABBIX_DATABASE;

### Step 8  
#### Start the Zabbix server
**Comand**

	sudo service zabbix-server start

**Script.**

	echo "------------------------------------------"
	echo "    8. Starting Zabbix Server        "
	echo "------------------------------------------"
	sudo service zabbix-server start;
	
### Step 9 
#### Optional to reduce the history table.

Additionally you can update the items table and set the item history table record to a fewer days.
**Comand**

	UPDATE items SET history = '15' WHERE history > '30';
