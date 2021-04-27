# DB2 Warehouse and encrypted backup

Encrypted backup cannot be immediately restored in a separate environment. Below I'm describing a practical steps how to do that.

In this article, the DB2 backup of BLUDB database is stored in /mnt/backup/BLUDB.0.db2inst1.DBPART000.20210425102241.001

## Verify that backup is encrypted

> db2ckbkp -H /mnt/backup/BLUDB.0.db2inst1.DBPART000.20210425102241.001 
```
	Platform                       -- 0x1E (Linux-x86-64)
	Encrypt Info Flags             -- 0x1
	                                  Source DB was encrypted
```

## Identify keystore and the secret key label

> db2 "restore db bludb from /mnt/backup taken at 20210425102241   encropts 'show master key details'"

The result of this command is a tricky one. Identify the diagnostic directory.
<br>
> db2 get dbm cfg | grep -i DIAGPATH
```
 Diagnostic data directory path               (DIAGPATH) = /mnt/blumeta0/db2/log/ $N
 Current member resolved DIAGPATH                        = /mnt/blumeta0/db2/log/NODE0000/
 Alternate diagnostic data directory path (ALT_DIAGPATH) = 
 Current member resolved ALT_DIAGPATH                    = 
```

> cd /mnt/blumeta0/db2/log/NODE0000/<br>

The location of keystore and label is stored in a specific file. Important: this data is related to the source database. 
<br>
> cat BLUDB.0.db2inst1.DBPART000.20210425102241.masterKeyDetails 
```
                 KeyStore Type: PKCS12
             KeyStore Location: /mnt/blumeta0/db2/keystore/keystore.p12
            KeyStore Host Name: thinkde
           KeyStore IP Address: 127.0.1.1
      KeyStore IP Address Type: IPV4
          Encryption Algorithm: AES
     Encryption Algorithm Mode: CBC
         Encryption Key Length: 256
              Master Key Label: DB2_SYSGEN_db2inst1_BLUDB_2021-03-19-13.57.02_A8CF4EED
```
## Extract master secret key from the source database.

List the content of source keystore.<br>

> gsk8capicmd_64 -cert - -list -db /mnt/blumeta0/db2/keystore/keystore.p12  -stashed
```
Certificates found
* default, - personal, ! trusted, # secret key
 #	DB2_SYSGEN_db2inst1_BLUDB_2021-03-19-13.57.02_A8CF4EED
```
The master key label is the same as report by *'show master key details'*

Export the key and move to the target DB2 host.<br>

> gsk8capicmd_64 -cert -export -db /tmp/keystore/keystore.p12 -stashed -label DB2_SYSGEN_db2inst1_BLUDB_2021-03-19-13.57.02_A8CF4EED -target secretkey.p12
```
Target database password :
```
The *Target password* is the key use to encrypt *secretkey.p12*

## Create the keystore in the target DB2 instance.

Assume the keystore is stored in */mnt/blumeta0/db2/keystore/*. Create directory, keystore and import *secretkey.p12* using the same master key label.<br>

> mkdir -p  /mnt/blumeta0/db2/keystore/<br>
> gsk8capicmd_64 -keydb -create -db /mnt/blumeta0/db2/keystore/keystore.p12   -pw "secret" -type pkcs12 -stash<br>
> gsk8capicmd_64 -cert -import  -db  /mnt/backup/secretkey.p12 -pw "secret"  -target   /mnt/blumeta0/db2/keystore/keystore.p12 -stashed -label DB2_SYSGEN_db2inst1_BLUDB_2021-03-19-13.57.02_A8CF4EED  <br>

Update DB2 Warehouse instance.<br>

> db2 update dbm cfg using keystore_location  /mnt/blumeta0/db2/keystore/keystore.p12  keystore_type pkcs12<br>

## Restore encrypted backup

> db2 restore db bludb from /mnt/backup taken at 20210425102241 no encrypt<br>

