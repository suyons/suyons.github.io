---
title: "Installing Oracle Database 19c on Ubuntu with Docker"
date: 2025-09-17
draft: false
tags: ["oracle", "docker", "ubuntu", "database"]
categories: ["DevOps"]
description: "A step-by-step guide to installing Oracle Database 19c on Ubuntu using Docker, including PDB management and data migration via dump files."
showToc: true
---

## Oracle Database 19c Requirements

- Oracle Database 19c officially supports only **Windows x64** and **Red Hat-based Linux x64** distributions.
- Other Linux distributions like Ubuntu are not officially supported.
- Docker images must be built on a supported OS.

## Preparing Required Files

### Building the Docker Image

Oracle does not distribute a pre-built Docker image — only a Dockerfile and installation scripts are provided.

[Oracle Database Docker GitHub Repository](https://github.com/oracle/docker-images/tree/main/OracleDatabase/19.3.0)

Clone the repository:

```bash
git clone https://github.com/oracle/docker-images.git
```

### Downloading the Oracle Database 19c Installer

Download the Oracle Database 19c installer from the official Oracle website:

[Oracle Database Software Downloads](https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html)

The target is `Oracle Database 19c (19.3) for Linux x86-64`, and the filename is `LINUX.X64_193000_db_home.zip`.

After downloading, transfer it to your Ubuntu server's home directory via SCP:

```bash
scp LINUX.X64_193000_db_home.zip docker@ubuntu:~
```

## Building the Docker Image

Move the downloaded file into the `docker-images/OracleDatabase/19.3.0` directory:

```bash
mv LINUX.X64_193000_db_home.zip docker-images/OracleDatabase/19.3.0/
```

The directory should contain the following files:

```bash
docker@ubuntu:~/docker-images/OracleDatabase/SingleInstance/dockerfiles/19.3.0$ ll
total 2988116
drwxrwxr-x  2 docker docker       4096 Sep 17 19:01 ./
drwxrwxr-x 10 docker docker       4096 Sep 17 18:58 ../
-rwxrwxr-x  1 docker docker       2727 Sep 17 18:58 checkDBStatus.sh*
-rwxrwxr-x  1 docker docker        904 Sep 17 18:58 checkSpace.sh*
-rw-rw-r--  1 docker docker         63 Sep 17 18:58 Checksum.ee
-rw-rw-r--  1 docker docker         65 Sep 17 18:58 Checksum.ee.arm64
-rw-rw-r--  1 docker docker         63 Sep 17 18:58 Checksum.se2
-rw-rw-r--  1 docker docker       7453 Sep 17 18:58 configTcps.sh
-rwxrwxr-x  1 docker docker       9047 Sep 17 18:58 createDB.sh*
-rw-rw-r--  1 docker docker       1567 Sep 17 18:58 createObserver.sh
-rw-rw-r--  1 docker docker       9204 Sep 17 18:58 dbca.rsp.tmpl
-rw-rw-r--  1 docker docker       6878 Sep 17 18:58 db_inst.rsp
-rw-rw-r--  1 docker docker       5077 Sep 17 18:58 Dockerfile
-rwxrwxr-x  1 docker docker       2712 Sep 17 18:58 installDBBinaries.sh*
-rw-rw-r--  1 docker docker 3059705302 Sep 17 18:57 LINUX.X64_193000_db_home.zip
-rw-rw-r--  1 docker docker       2013 Sep 17 18:58 relinkOracleBinary.sh
-rwxrwxr-x  1 docker docker      11283 Sep 17 18:58 runOracle.sh*
-rwxrwxr-x  1 docker docker       1021 Sep 17 18:58 runUserScripts.sh*
-rwxrwxr-x  1 docker docker       1141 Sep 17 18:58 setPassword.sh*
-rwxrwxr-x  1 docker docker        679 Sep 17 18:58 setupLinuxEnv.sh*
-rwxrwxr-x  1 docker docker        679 Sep 17 18:58 startDB.sh*
```

Build the Docker image:

```bash
cd docker-images/OracleDatabase/SingleInstance/dockerfiles
./buildContainerImage.sh -v 19.3.0 -e
# ... output omitted ...
Oracle Database container image for 'ee' version 19.3.0 is ready to be extended:

  --> oracle/database:19.3.0-ee

Build completed in 110 seconds.
```

Confirm the image was created:

```bash
docker@ubuntu:~$ docker images
REPOSITORY        TAG         IMAGE ID       CREATED          SIZE
oracle/database   19.3.0-ee   a4dcf85abc5b   22 minutes ago   6.54GB
```

## Creating and Starting the Container

```bash
docker run --name oracle19c -p 1521:1521 -p 5500:5500 \
-e ORACLE_PDB=orapdb1 \
-e ORACLE_PWD='YOUR_PASSWORD_HERE!' \
-e ORACLE_MEM=3000 \
-v /opt/oracle/oradata:/opt/oracle/oradata \
-d oracle/database:19.3.0-ee
```

### Parameter Reference

| Parameter | Description |
|---|---|
| `--name oracle19c` | Container name |
| `-p 1521:1521` | Oracle Listener port |
| `-p 5500:5500` | Enterprise Manager Express port |
| `-e ORACLE_PDB=orapdb1` | PDB name (becomes the SID for connections) |
| `-e ORACLE_PWD='...'` | System user password (use single quotes if it contains special characters) |
| `-e ORACLE_MEM=3000` | Memory allocation in MB |
| `-v /opt/oracle/oradata:...` | Mounts a host directory as the data store |
| `-d` | Run in background |

Verify the container is running:

```bash
docker@ubuntu:~$ docker ps
CONTAINER ID   IMAGE                       COMMAND       CREATED         STATUS                         PORTS   NAMES
cba62376dcd2   oracle/database:19.3.0-ee   "/bin/bash…"  About a minute  Up About a minute (starting)   ...     oracle19c
```

### Troubleshooting Initial Startup Errors

Check the container logs:

```bash
docker@ubuntu:~$ docker logs -f oracle19c
# ... output omitted ...
mkdir: cannot create directory '/opt/oracle/oradata/dbconfig': Permission denied
# ... more errors ...
#####################################
########### E R R O R ###############
DATABASE SETUP WAS NOT SUCCESSFUL!
Please check output for further info!
########### E R R O R ###############
#####################################
```

Even with errors in the log, try connecting to check the actual database state:

```bash
docker@ubuntu:~$ docker exec -it oracle19c bash
bash-4.2$ sqlplus / as sysdba
```

```sql
SQL> select status from v$instance;

STATUS
------------
MOUNTED
```

The database is mounted but not open. Trying to open it reveals a missing control file:

```sql
SQL> alter database open;
ERROR at line 1:
ORA-00210: cannot open the specified control file
ORA-00202: control file: '/opt/oracle/cfgtoollogs/dbca/ORCLCDB/tempControl.ctl'
```

The root cause: the `/opt/oracle/oradata` directory is owned by `root:root`, so the Oracle process can't write to it.

Fix it from the host (UID `54321` / GID `54322` are the `oracle:dba` IDs inside the container):

```bash
docker@ubuntu:~$ sudo chown -R 54321:54322 /opt/oracle/oradata
docker@ubuntu:~$ docker restart oracle19c
```

After restarting, the logs should show:

```
#########################
DATABASE IS READY TO USE!
#########################
```

Verify the database state:

```sql
SQL> select status from v$instance;

STATUS
------------
OPEN
```

```sql
SQL> select name, open_mode from v$pdbs;

NAME         OPEN_MODE
------------ ----------
PDB$SEED     READ ONLY
ORAPDB1      READ WRITE
```

The PDB is open and ready.

## Connecting to Oracle Database

Use a GUI client like DBeaver or SQL Developer with the following connection details:

| Field | Value |
|---|---|
| Host | Ubuntu server IP |
| Port | 1521 |
| Service name | ORAPDB1 |
| Username | SYSTEM |
| Password | (password set during container creation) |

![DBeaver connection](/images/oracle-19c-ubuntu-docker/image.png)

## PDB Management

### Rename a PDB

```sql
-- 1. Close the PDB
ALTER PLUGGABLE DATABASE ORAPDB1 CLOSE;

-- 2. Open in restricted mode
ALTER PLUGGABLE DATABASE ORAPDB1 OPEN RESTRICTED;

-- 3. Rename
ALTER PLUGGABLE DATABASE ORAPDB1 RENAME GLOBAL_NAME TO NEW_PDB_NAME;

-- 4. Close and reopen
ALTER PLUGGABLE DATABASE NEW_PDB_NAME CLOSE;
ALTER PLUGGABLE DATABASE NEW_PDB_NAME OPEN;

-- Verify
SELECT name, open_mode FROM v$pdbs;
```

### Create a PDB

```sql
CREATE PLUGGABLE DATABASE newpdb ADMIN USER pdbadmin IDENTIFIED BY "password!";
```

> Note on quotes: use single quotes in bash shell, double quotes in SQL*Plus when the password contains special characters.

### Drop a PDB

```sql
ALTER PLUGGABLE DATABASE pdbname CLOSE;
DROP PLUGGABLE DATABASE pdbname INCLUDING DATAFILES;
```

### Open / Close a PDB

```sql
ALTER PLUGGABLE DATABASE pdbname OPEN;
ALTER PLUGGABLE DATABASE pdbname CLOSE;
```

## Migrating Data with a Dump File

If you have an existing Oracle 19c instance and want to migrate its data to this new Docker container, use `expdp`/`impdp`.

### Export the Dump File

On the source server:

```bash
[oracle@localhost ~]$ expdp USERNAME/'PASSWORD!'@PDB_NAME \
  schemas=SCHEMA_NAME \
  directory=DATA_PUMP_DIR \
  dumpfile=dumpfile.dmp \
  logfile=export.log

# ... output omitted ...
Job "SYSTEM"."SYS_EXPORT_SCHEMA_01" successfully completed at Wed Sep 17 23:12:44 2025 elapsed 0 00:00:33
```

Transfer the dump file to the Ubuntu host:

```bash
scp LINUX.X64_193000_db_home.zip docker@ubuntu:~
```

### Find the DATA_PUMP_DIR Path

Connect to the container and query the directory path:

```sql
bash-4.2$ sqlplus system/'MY_PASSWORD_HERE!'

SQL> SELECT directory_name, directory_path FROM dba_directories WHERE directory_name = 'DATA_PUMP_DIR';

DIRECTORY_NAME       DIRECTORY_PATH
-------------------- -------------------------------------------------------
DATA_PUMP_DIR        /opt/oracle/admin/ORCLCDB/dpdump/3F009C70B6160238E063...
```

### Copy the Dump File Into the Container

```bash
docker cp ~/dumpfile.dmp oracle19c:/opt/oracle/admin/ORCLCDB/dpdump/3F009C70B6160238E063020011AC8843/
Successfully copied 245MB to oracle19c:/opt/oracle/admin/ORCLCDB/dpdump/3F009C70B6160238E063020011AC8843/
```

### Create the Tablespace

The tablespace from the source database must exist before importing:

```sql
CREATE TABLESPACE USERS_DATA
DATAFILE '/opt/oracle/oradata/ORCLCDB/ORAPDB1/users_data01.dbf'
SIZE 100M AUTOEXTEND ON NEXT 50M MAXSIZE UNLIMITED;
```

### Import the Dump File

```bash
docker@ubuntu:~$ docker exec -it oracle19c bash
bash-4.2$ impdp system/'MY_PASSWORD_HERE!' \
  schemas=SCHEMA_NAME \
  directory=DATA_PUMP_DIR \
  dumpfile=dumpfile.dmp \
  logfile=import.log

# ... output omitted ...
Job "SYSTEM"."SYS_IMPORT_SCHEMA_01" completed with 1 error(s) at Wed Sep 17 14:49:51 2025 elapsed 0 00:00:06
```

## References

- [Oracle Database Docker GitHub Repository](https://github.com/oracle/docker-images/tree/main/OracleDatabase)
- [Oracle Database Software Downloads](https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html)
- [Oracle Database 19c Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/19/index.html)
