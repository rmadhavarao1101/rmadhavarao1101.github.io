----------------
Resolving an Oracle ODA DNS Mismatch: A Step-by-Step Guide
----------------
<h2 class="heading-blue">Problem</h2>
When administering an Oracle Database Appliance (ODA), you may update the OS-level DNS servers (for example, /etc/resolv.conf) but find that odacli describe-system still reports the old DNS server add[...] 

<h2 class="heading-orange">Important disclaimer</h2>
Modifying ODA internal databases directly is risky. Only proceed after:
- Taking a full backup (or at minimum backing up affected tables).
- Consulting Oracle support / your change management process if this is a production system.
- Performing these steps during a maintenance window if possible.

This guide documents a careful, step-by-step approach — ensure you understand each command before executing it.

<h2 class="heading-blue">Prerequisites</h2>
- Root or equivalent access to the ODA appliance.
- Access to the internal dcsagentdb MySQL instance (the DCS agent DB).
- Familiarity with basic MySQL commands and SQL transactions.
- Backup plan in place.

<h2 class="step-heading">Step 1 — Verify the current ODA configuration</h2>
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

<h2 class="step-heading">Step 2 — Confirm OS-level DNS is updated</h2>
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

<h2 class="step-heading">Step 3 — Access the ODA internal MySQL (dcsagentdb)</h2>
Connect to the appliance's internal MySQL instance using the supplied MySQL binary and defaults file:

```sh
# /opt/oracle/dcs/mysql/bin/mysql --defaults-file=/opt/oracle/dcs/mysql/etc/mysqldb.cnf
Welcome to the MySQL monitor.  Commands end with ; or \g.
mysql>
```

<h2 class="step-heading">Step 4 — Locate the appliance’s system ID</h2>
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

<h2 class="step-heading">Step 5 — Inspect the DNS entries recorded internally</h2>
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

<h2 class="step-heading">Step 6 — Back up the table before changes</h2>
Always take a backup of the table you plan to modify:

```sql
mysql> CREATE TABLE SysInstance_dnsServers_BACKUP_20251203 AS SELECT * FROM SysInstance_dnsServers;
Query OK, 3 rows affected (0.01 sec)
```

Adjust the backup table name (`YYYYMMDD`) to match the date/time you perform this operation.

<h2 class="step-heading">Step 7 — Insert the new DNS entries</h2>
Add the correct DNS server IPs into the table:

```sql
mysql> INSERT INTO SysInstance_dnsServers VALUES ('ddummy9-000f-14ha-oracl-jkd8dj3jk29d39','10.15.0.1');
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO SysInstance_dnsServers VALUES ('ddummy9-000f-14ha-oracl-jkd8dj3jk29d39','10.15.0.2');
Query OK, 1 row affected (0.00 sec)
```

<h2 class="step-heading">Step 8 — Remove the old / stale DNS entries</h2>
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

<h2 class="step-heading">Step 9 — Verify the table now contains only the new IPs</h2>
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

<h2 class="step-heading">Step 10 — Validate via odacli</h2>
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

<h2 class="heading-orange">Post-change checks and best practices</h2>
- Keep the backup table (created earlier) for a reasonable retention period before dropping it.
- Log the change (who, when, justification, commands run).
- If possible, replicate the steps in a non-production environment first.
- Consider opening a support ticket with Oracle for guidance — direct DB edits are generally a last resort.

<h2 class="heading-blue">Summary</h2>
When odacli reports outdated DNS servers while the OS uses updated resolvers, the root cause is typically a mismatch between the OS and the ODA internal DCS agent DB. Carefully updating the SysInstanc[...] 

<h2 class="heading-muted">References and further reading</h2>
- Oracle Database Appliance documentation (refer to your version-specific guides / Doc ID 2796053.1).
- Oracle support advisories — consult My Oracle Support for known issues and best practices.

<!-- Editor: GitHub Copilot Chat Assistant -->
