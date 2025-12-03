----------------
Resolving an Oracle ODA DNS Mismatch: A Step-by-Step Guide
----------------

Problem
-------
When administering an Oracle Database Appliance (ODA), you may update the OS-level DNS servers (for example, /etc/resolv.conf) but find that odacli describe-system still reports the old DNS server addresses. This indicates a mismatch between the OS network configuration and the ODA internal metadata (the DCS agent database). When ODA management commands do not correctly reflect network changes, you can manually update the internal database to restore consistency.

Important disclaimer
--------------------
Modifying ODA internal databases directly is risky. Only proceed after:
- Taking a full backup (or at minimum backing up affected tables).
- Consulting Oracle support / your change management process if this is a production system.
- Performing these steps during a maintenance window if possible.

This guide documents a careful, step-by-step approach — ensure you understand each command before executing it.

Prerequisites
-------------
- Root or equivalent access to the ODA appliance.
- Access to the internal dcsagentdb MySQL instance (the DCS agent DB).
- Familiarity with basic MySQL commands and SQL transactions.
- Backup plan in place.

Step 1 — Verify the current ODA configuration
---------------------------------------------
Run odacli to see what the appliance reports:

```sh
# odacli describe-system
Appliance Information
----------------------------------------------------------------
              ID: ddummy9-000f-14ha-oracl-jkd8dj3jk29d39
           Platform: X8-2M

System Information
----------------------------------------------------------------
            Name: odadb01
     Domain Name: overpass.net
       Time Zone: America/NewYork
       DB Edition: EE
       DNS Servers: 10.5.4.6 10.5.1.3 10.5.4.7   # old/stale entries
       NTP Servers: 10.5.1.8
```

Step 2 — Confirm OS-level DNS is updated
----------------------------------------
Check /etc/resolv.conf to ensure the OS is using the new DNS servers. Editing was likely performed already:

```sh
# cat /etc/resolv.conf
search bridge.net
#nameserver 10.5.4.6   # old, commented out
#nameserver 10.5.4.7
#nameserver 10.5.1.3
nameserver 10.15.0.1   # new
nameserver 10.15.0.2
```

Validate name resolution:

```sh
# nslookup odadb01
Server:         10.15.0.1
Address:        10.15.0.1#53
Name:           odadb01.bridge.net
Address:        10.4.1.1
```

If OS-level lookups are working but odacli still shows old DNS entries, proceed to update the internal database.

Step 3 — Access the ODA internal MySQL (dcsagentdb)
---------------------------------------------------
Connect to the appliance's internal MySQL instance using the supplied MySQL binary and defaults file:

```sh
# /opt/oracle/dcs/mysql/bin/mysql --defaults-file=/opt/oracle/dcs/mysql/etc/mysqldb.cnf
Welcome to the MySQL monitor.  Commands end with ; or \g.
mysql>
```

Step 4 — Locate the appliance’s system ID
-----------------------------------------
Switch to the dcsagentdb and identify the SysInstance entry for this appliance:

```sql
mysql> USE dcsagentdb;
mysql> SELECT id, name FROM SysInstance;
+--------------------------------------+-----------+
| id                                   | name      |
+--------------------------------------+-----------+
| ddummy9-000f-14ha-oracl-jkd8dj3jk29d39 | odadb01 |
+--------------------------------------+-----------+
1 row in set (0.00 sec)
```

Make a note of the `id` value (the unique system ID) — you will use it in subsequent queries.

Step 5 — Inspect the DNS entries recorded internally
----------------------------------------------------
View the current entries in the SysInstance_dnsServers table for the appliance:

```sql
mysql> SELECT * FROM SysInstance_dnsServers WHERE SysInstance_id = 'ddummy9-000f-14ha-oracl-jkd8dj3jk29d39';
+--------------------------------------+------------+
| SysInstance_id                       | dnsServers |
+--------------------------------------+------------+
| ddummy9-000f-14ha-oracl-jkd8dj3jk29d39 | 10.5.4.6  |
| ddummy9-000f-14ha-oracl-jkd8dj3jk29d39 | 10.5.1.3  |
| ddummy9-000f-14ha-oracl-jkd8dj3jk29d39 | 10.5.4.7  |
+--------------------------------------+------------+
3 rows in set (0.00 sec)
```

Step 6 — Back up the table before changes
-----------------------------------------
Always take a backup of the table you plan to modify:

```sql
mysql> CREATE TABLE SysInstance_dnsServers_BACKUP_20251203 AS SELECT * FROM SysInstance_dnsServers;
Query OK, 3 rows affected (0.01 sec)
```

Adjust the backup table name (`YYYYMMDD`) to match the date/time you perform this operation.

Step 7 — Insert the new DNS entries
-----------------------------------
Add the correct DNS server IPs into the table:

```sql
mysql> INSERT INTO SysInstance_dnsServers VALUES ('ddummy9-000f-14ha-oracl-jkd8dj3jk29d39','10.15.0.1');
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO SysInstance_dnsServers VALUES ('ddummy9-000f-14ha-oracl-jkd8dj3jk29d39','10.15.0.2');
Query OK, 1 row affected (0.00 sec)
```

Step 8 — Remove the old / stale DNS entries
-------------------------------------------
Delete the outdated IP addresses from the table:

```sql
mysql> DELETE FROM SysInstance_dnsServers
       WHERE SysInstance_id = 'ddummy9-000f-14ha-oracl-jkd8dj3jk29d39'
         AND dnsServers = '10.5.4.6';
Query OK, 1 row affected (0.00 sec)

mysql> DELETE FROM SysInstance_dnsServers
       WHERE SysInstance_id = 'ddummy9-000f-14ha-oracl-jkd8dj3jk29d39'
         AND dnsServers = '10.5.4.7';
Query OK, 1 row affected (0.00 sec)

mysql> DELETE FROM SysInstance_dnsServers
       WHERE SysInstance_id = 'ddummy9-000f-14ha-oracl-jkd8dj3jk29d39'
         AND dnsServers = '10.5.1.3';
Query OK, 1 row affected (0.00 sec)
```

Step 9 — Verify the table now contains only the new IPs
-------------------------------------------------------
Confirm the database reflects the intended configuration:

```sql
mysql> SELECT * FROM SysInstance_dnsServers WHERE SysInstance_id = 'ddummy9-000f-14ha-oracl-jkd8dj3jk29d39';
+--------------------------------------+------------+
| SysInstance_id                       | dnsServers |
+--------------------------------------+------------+
| ddummy9-000f-14ha-oracl-jkd8dj3jk29d39 | 10.15.0.1 |
| ddummy9-000f-14ha-oracl-jkd8dj3jk29d39 | 10.15.0.2 |
+--------------------------------------+------------+
2 rows in set (0.00 sec)
```

Step 10 — Validate via odacli
-----------------------------
Exit MySQL and run odacli describe-system again to confirm the appliance now reports the updated DNS servers:

```sh
# odacli describe-system
Appliance Information
----------------------------------------------------------------
...
System Information
----------------------------------------------------------------
            Name: odadb01
     Domain Name: overpass.net
       Time Zone: America/Halifax
       DB Edition: EE
       DNS Servers: 10.15.0.1 10.15.0.2   # Updated
       NTP Servers: 10.5.1.8
...
```

If odacli still shows stale values:
- Recheck the DB table contents to ensure changes were committed.
- Confirm there are no cached copies or management agents that need a restart.
- Consult Oracle support if the appliance does not reflect the database changes after verification.

Post-change checks and best practices
------------------------------------
- Keep the backup table (created earlier) for a reasonable retention period before dropping it.
- Log the change (who, when, justification, commands run).
- If possible, replicate the steps in a non-production environment first.
- Consider opening a support ticket with Oracle for guidance — direct DB edits are generally a last resort.

Summary
-------
When odacli reports outdated DNS servers while the OS uses updated resolvers, the root cause is typically a mismatch between the OS and the ODA internal DCS agent DB. Carefully updating the SysInstance_dnsServers table in dcsagentdb synchronizes the internal metadata and resolves the discrepancy. Always back up the table, document the change, and consult Oracle support for production-critical systems.

References and further reading
------------------------------
- Oracle Database Appliance documentation (refer to your version-specific guides / Doc ID 2796053.1).
- Oracle support advisories — consult My Oracle Support for known issues and best practices.

<!-- Editor: GitHub Copilot Chat Assistant -->
