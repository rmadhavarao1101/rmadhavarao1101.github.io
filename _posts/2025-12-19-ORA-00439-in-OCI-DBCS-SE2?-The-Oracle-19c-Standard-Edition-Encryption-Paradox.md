## ORA-00439 in OCI DBCS SE2? The Oracle 19c Standard Edition Encryption Paradox

Introduction: The TDE Requirement in Oracle Cloud
----

If you're managing Oracle databases on Oracle Cloud Infrastructure (OCI) Database Cloud Service (DBCS), you know that security is paramount. For compliance reasons, OCI enables Transparent Data Encryption (TDE) by default for all provisioned databases—even those running the cost-effective Standard Edition 2 (SE2).

This is where the paradox begins. TDE is historically an Enterprise Edition (EE) feature. While OCI allows existing tablespaces to be encrypted on SE2, we often encounter a confusing error when trying to create a new tablespace, despite the database wallet being open and existing data encrypted.

This post details a specific bug that causes the dreaded ORA-00439 error in 19c SE2 environments and provides the definitive solution.

1. Environment and Initial Encryption Check
---   
We are running a typical OCI DBCS Standard Edition 2 instance.

A. Database Version
--

```sql

SQL> SELECT BANNER_FULL FROM v$version;

BANNER_FULL
--------------------------------------------------------------------------------
Oracle Database 19c Standard Edition 2 Release 19.0.0.0.0 - Production
Version 19.21.0.0.0

B. Tablespace Encryption Status
--

The dba_tablespaces view confirms that our core tablespaces were created with TDE enabled (ENCRYPTED = YES):

```sql

SQL> select tablespace_name,ENCRYPTED from dba_tablespaces;

TABLESPACE_NAME                ENC
------------------------------ ---
SYSTEM                         YES
SYSAUX                         YES
UNDOTBS1                       YES
USERS                          YES
TEMP                           YES

C. TDE Wallet Status
--

The encryption wallet is clearly open and functional:

```sql

SQL> SET LINESIZE 200
SQL> COLUMN wrl_parameter FORMAT A50 
SQL> SELECT * FROM v$encryption_wallet;

WRL_TYPE             WRL_PARAMETER                                      STATUS                         WALLET_TYPE          WALLET_OR KEYSTORE FULLY_BAC     CON_ID
-------------------- -------------------------------------------------- ------------------------------ -------------------- --------- -------- --------- ----------
FILE                 /opt/oracle/dcs/commonstore/wallets/vcccdb_rtn_yyz OPEN                           AUTOLOGIN            SINGLE    NONE     NO                 1
                     /tde/
... (other entries)
The expectation: Since encryption is mandated, active, and the wallet is open, creating a new tablespace should work and be encrypted by default.

2. The Failure: ORA-00439
--

When attempting to create a new tablespace (TEST), we hit an unexpected wall:

```sql

SQL> CREATE TABLESPACE TEST
DATAFILE '+DATA' SIZE 256M 
AUTOEXTEND ON NEXT 256M MAXSIZE UNLIMITED;
CREATE TABLESPACE TEST
*
ERROR at line 1:
ORA-00439: feature not enabled: Transparent Data Encryption
The Alert Log confirms the error:

2025-12-18T07:41:05.176777-08:00
CREATE TABLESPACE TEST
DATAFILE '+DATA' SIZE 256M
AUTOEXTEND ON NEXT 256M MAXSIZE UNLIMITED
ORA-439 signalled during: CREATE TABLESPACE TEST
DATAFILE '+DATA' SIZE 256M
AUTOEXTEND ON NEXT 256M MAXSIZE UNLIMITED...

This error is highly confusing: How can TDE be not enabled when the tablespaces are encrypted and the wallet is open?

3. The Root Cause: Oracle Bug 37740291
---

The problem lies not in your configuration, but in a known defect within the Standard Edition 2 codebase concerning TDE validation in cloud environments.

The root cause is identified as:

Bug ID: 37740291

Description: basedb: standard edition tablespace creation fails with ORA-439: feature not enabled: transparent data encryption

Oracle Support Document: KI40518

This bug affects several versions, including the 19c Release Updates, up to and including 19.27.0. The SE2 code incorrectly validates the TDE feature when creating a new tablespace, leading to the ORA-00439 error.

Note: Oracle has explicitly stated that for this bug, the WORKAROUND is NONE. The only way to resolve this is by applying the required patch.
---

4. The Resolution: Applying the DB Release Update
---
To fix this specific issue, you must apply the Database Release Update (RU) that includes the patch for Bug 37740291.

A. The Fix Version
--
According to Oracle Support Document KI40518, the fix is first included in:

19.28.0.0.250715 (July 2025) DB Release Update (DB RU)

B. Post-Patch Validation
--

After applying the necessary patch (which in this scenario took the database to 19.28.0.0.0), the database version is updated:

```sql

SQL> select banner_full from v$version;

BANNER_FULL
--------------------------------------------------------------------------------
Oracle Database 19c Standard Edition 2 Release 19.0.0.0.0 - Production
Version 19.28.0.0.0
C. The Successful Tablespace Creation
With the patched version, the tablespace creation command now executes successfully, as expected:

```sql


SQL> CREATE TABLESPACE TEST
DATAFILE '+DATA' SIZE 256M 
AUTOEXTEND ON NEXT 256M MAXSIZE UNLIMITED;

Tablespace created.

The final check confirms that the new tablespace is also created with default encryption enabled, maintaining OCI's security posture:

```sql

SQL> select tablespace_name,ENCRYPTED from dba_tablespaces;

TABLESPACE_NAME                ENC
------------------------------ ---
SYSTEM                         YES
SYSAUX                         YES
UNDOTBS1                       YES
USERS                          YES
TEMP                           YES
TEST                           YES

6 rows selected.

Conclusion
----

Encountering the ORA-00439 TDE error in an Oracle Cloud SE2 environment is confusing, but it's important to remember that this behavior is often due to product-specific bugs related to cloud feature enablement, not configuration errors.

If you are running Oracle 19c SE2 on OCI and face this error during tablespace creation, the definitive solution is to patch your database to the 19.28.0.0.0 RU or higher to incorporate the fix for Bug 37740291.

Keeping your DBCS instances updated is crucial for maintaining security and stability, especially when dealing with features like TDE that bridge the gap between Standard and Enterprise Edition capabilities in a cloud context.
