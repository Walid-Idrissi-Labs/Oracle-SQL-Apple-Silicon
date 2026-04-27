# Oracle SQL on macOS ARM
 
Running Oracle Database, SQL\*Plus, and SQL Developer on Apple Silicon (M1/M2/M3).
The official Oracle XE installer is x86-only. This guide uses Oracle's official container image running under x86 emulation via Rosetta 2, with native ARM64 builds for the client tools.
 
Tested on macOS with an M1 chip. SQL\*Plus version: `23.26.1.0.0`.
 
---

## 1. Oracle Database (Docker)
 
The database runs inside a Docker container. Oracle publishes an official image through their container registry. You need to authenticate before pulling it.
 
**Login to Oracle Container Registry**
 
```bash
docker login container-registry.oracle.com
```
 
You will be prompted for your Oracle account credentials. If you do not have an auth token, generate one after logging into the registry website and save it somewhere safe.
 
The registry page for the image: `container-registry.oracle.com` > Database > Free
 
**Pull the image**
 
```bash
docker pull container-registry.oracle.com/database/free:latest-lite
```
 
The `latest-lite` tag is a stripped-down image with a smaller footprint, suitable for development and coursework. If you run into missing database features later, switch to `latest` (the full image).
 
**Run the container**
 
```bash
docker run -d \
  --name oracle-free \
  -p 1521:1521 \
  -e ORACLE_PWD="mypassword" \
  container-registry.oracle.com/database/free:latest-lite
```
 
`-p 1521:1521` maps the container port to the host. This is the port you will use in every connection string. `ORACLE_PWD` sets the password for the `SYSTEM` and `SYS` accounts.
 
Wait for the database to finish initializing before connecting. Check with:
 
```bash
docker logs oracle-free
```
 
The database is ready when you see `DATABASE IS READY TO USE`.
 
To verify your password at any point:
 
```bash
docker inspect oracle-free | grep ORACLE_PWD
```
 
**Starting the container after a reboot**
 
The container persists its data across stops and starts. To bring it back up:
 
```bash
docker start oracle-free
```
 
> The PDB (Pluggable Database) in this image is named `FREEPDB1`. If you are following the official Oracle tutorial which references `XEPDB1`, substitute `FREEPDB1` everywhere. Using `XEPDB1` will throw an error since that PDB does not exist in this image.
 
---

## 2. SQL\*Plus and Oracle Instant Client 
 
Oracle ships native ARM64 builds of Instant Client. Do not use the x86 ZIP files from the standard tutorial.
 
Download the three packages from [Oracle's ARM64 Instant Client page](https://www.oracle.com/database/technologies/instant-client/macos-arm64-downloads.html): Basic, SQL\*Plus, and SDK.
 
Mount and install each one:
 
```bash
hdiutil mount ~/Downloads/instantclient-basic-macos.arm64-23.3.0.23.09.dmg
cd /Volumes/instantclient-basic-macos.arm64-23.3.0.23.09/
sh ./install_ic.sh
cd ~/Downloads/
hdiutil unmount /Volumes/instantclient-basic-macos.arm64-23.3.0.23.09/
 
hdiutil mount ~/Downloads/instantclient-sqlplus-macos.arm64-23.3.0.23.09.dmg
cd /Volumes/instantclient-sqlplus-macos.arm64-23.3.0.23.09/
sh ./install_ic.sh
cd ~/Downloads/
hdiutil unmount /Volumes/instantclient-sqlplus-macos.arm64-23.3.0.23.09/
 
hdiutil mount ~/Downloads/instantclient-sdk-macos.arm64-23.3.0.23.09.dmg
cd /Volumes/instantclient-sdk-macos.arm64-23.3.0.23.09/
sh ./install_ic.sh
cd ~
hdiutil unmount /Volumes/instantclient-sdk-macos.arm64-23.3.0.23.09/
```
 
Move the installed directory to a permanent location:
 
```bash
mkdir -p ~/oracle
mv ~/Downloads/instantclient_23_3/ ~/oracle/
```
 
**Configure environment variables**
 
Add the following to `~/.zshrc`:
 
```bash
# Oracle Instant Client
export ORACLE_HOME=/Users/[yourUsername]/oracle/instantclient_23_3
export DYLD_LIBRARY_PATH=$ORACLE_HOME:$DYLD_LIBRARY_PATH
export PATH=$ORACLE_HOME:$PATH
 
# Optional: for OCCI/C++ development
export OCI_LIB_DIR="$ORACLE_HOME"
export OCI_INC_DIR="$ORACLE_HOME/sdk/include"
```
 
 
```bash
source ~/.zshrc
```
 
 
**Connect to the database**
 
```bash
sqlplus system/mypassword@//localhost:1521/FREEPDB1
```

> Installation process based on [this YouTube video](https://www.youtube.com/watch?v=U_mFVr5vPas).
 
---


 ## 3. Create a User
 
Working as `SYSTEM` for everything is not good practice. Create a dedicated user inside the PDB.
 
Connect as `SYSTEM` first:
 
```bash
sqlplus system/mypassword@//localhost:1521/FREEPDB1
```
 
Then run:
 
```sql
ALTER SESSION SET CONTAINER = FREEPDB1;
 
CREATE USER myuser IDENTIFIED BY mypassword;
 
GRANT CONNECT, RESOURCE TO myuser;
```
 
For the tablespace quota, the standard command references the `USERS` tablespace, which does not exist in this image. To check what is available:
 
```sql
SELECT tablespace_name FROM dba_tablespaces;
```
 
Use `SYSAUX` instead of `USERS`:
 
```sql
ALTER USER myuser DEFAULT TABLESPACE SYSAUX QUOTA UNLIMITED ON SYSAUX;
```
 
This sets the default storage location for the user's objects and prevents `ORA-01950` errors when creating tables. It does not retroactively move any existing objects.
 
Verify the user works:
 
```bash
sqlplus myuser/mypassword@//localhost:1521/FREEPDB1
```
 
```sql
SELECT USER FROM DUAL;
```
 
Expected output:
 
```
USER
--------------------------------------------------------------------------------
MYUSER
```
 
---

## 4. SQL Developer Connection

| Field | Value |
|---|---|
| Connection Name | anything (e.g. `oracle-free`) |
| Username | `myuser` |
| Password | `mypassword` |
| Connection Type | Basic |
| Hostname | `localhost` |
| Port | `1521` |
| Service Name | `FREEPDB1` |
 
Select **Service Name**, not SID







