# Managing Backups through the Operator

FoundationDB has out-of-the-box support for backing up to an S3-compatible object store, and the operator supports this through a special resource type for backup and restore.
These backups run continuously, and simultaneously build new snapshots while backing up new mutations.
You can restore to any point in time after the end of the first snapshot.

You can find more information about the backup feature in the [FoundationDB Backup documentation](https://apple.github.io/foundationdb/backups.html).

**Warning**: Support for backups in the operator is still in development, and there are some missing features.

## Example Backup

This is a sample configuration for running a continuous backup of a cluster.

This sample assumes some configuration that you will need to fill in based on the details of your environment.
Those assumptions are explained in comments below.

```yaml
apiVersion: apps.foundationdb.org/v1beta2
kind: FoundationDBBackup
metadata:
  name: sample-cluster
spec:
  version: 7.1.26
  clusterName: sample-cluster
  blobStoreConfiguration:
    accountName: account@object-store.example:443
  podTemplateSpec:
    spec:
      volumes:
        - name: backup-credentials
          secret:
            secretName: backup-credentials
        - name: fdb-certs
          secret:
            secretName: fdb-certs
      containers:
        - name: foundationdb
          env:
            - name: FDB_BLOB_CREDENTIALS
              value: /var/backup-credentials/credentials
            - name: FDB_TLS_CERTIFICATE_FILE
              value: /var/fdb-certs/cert.pem
            - name: FDB_TLS_CA_FILE
              value: /var/fdb-certs/ca.pem
            - name: FDB_TLS_KEY_FILE
              value: /var/fdb-certs/key.pem
          volumeMounts:
            - name: fdb-certs
              mountPath: /var/fdb-certs
            - name: backup-credentials
              mountPath: /var/backup-credentials
---
apiVersion: v1
kind: Secret
metadata:
  name: backup-credentials
type: Opaque
stringData:
  credentials: |
    {
        "accounts": {
            "account@object-store.example": {
                "secret" : "YOUR_ACCOUNT_KEY"
            }
        }
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: fdb-certs
stringData:
  cert.pem: |
    # Put your certificate here.
  key.pem: |
    # Put your key here.
  ca.pem: |
    # Put your CA file here.
```

Creating this resource will tell the operator to do the following things:

1. Create a `sample-cluster-backup-agents` deployment running FoundationDB backup agent processes connecting to the cluster.
2. Run an `fdbbackup start` command to start a backup at `https://object-store.example:443/sample-cluster` using the bucket name `fdb-backups`.

Do note, that if a port is not provided in the `blobStoreConfiguration.accountName`, it will default to `443`,
or `80` if `secure_connection` is disabled.

## Using Secure Connections to the Object Store

By default, the operator assumes you want to use secure connections to your object store. In order to do this, you must provide a certificate, key, and CA file to the backup agents. The CA file must contain the root CA for your object store. The certificate and key must be parseable in order to initialize the TLS subsystem in the backup agents, but the agents will not use the certificate and key to communicate with the object store. You can configure the paths to these files through the environment variables `FDB_TLS_CERTIFICATE_FILE`, `FDB_TLS_KEY_FILE`, and `FDB_TLS_CA_FILE`. In the example above, we have all three of these defined in a secret called `fdb-certs`.

If you are configuring your cluster to use TLS for connections within the cluster, the backup agents will use the same certificate, key, and CA file for the connections to the cluster, so you must make sure the configuration is valid for this purpose as well.

## Configuring Your Account

Before you start a backup, you will need to configure an account in your object store. Depending on the implementation details of your object store, you may also need to configure a bucket in advance, but the FDB backup process will attempt to automatically create one. You can specify the bucket name in the `bucket` field of the backup spec. In the example above, we have an account called `account` at the object store `https://object-store.example`, and it has a bucket called `fdb-backups`.

You will need to expose the password or account key for the object store account through a credentials file. The format of the credentials file is defined in the FoundationDB backup documentation. You need to expose this credentials file to the backup agents, as shown in the example above. You can configure the path to the credentials file through the `FDB_BLOB_CREDENTIALS` environment variable.

## Configuring additional URL parameters

FoundationDB supports [URL parameters](https://apple.github.io/foundationdb/backups.html#backup-urls) those can be specified as a `map[string]string` in the `blobStoreConfiguration`.

If you want to disable the secure connection e.g. for testing you can set the `secure_connection` to `0`:

```yaml
apiVersion: apps.foundationdb.org/v1beta2
kind: FoundationDBBackup
metadata:
  name: sample-cluster
spec:
  version: 7.1.26
  clusterName: sample-cluster
  blobStoreConfiguration:
    accountName: account@object-store.example:443
    urlParameters:
    - "secure_connection=0"
```

## Configuring the Operator

The operator will run `fdbbackup` commands to manage the backup, so the operator needs to have access to the object store as well.
You can configure that access the same way as you do for the backup agents, by defining the environment variables `FDB_BLOB_CREDENTIALS`, `FDB_TLS_CERTIFICATE_FILE`, `FDB_TLS_KEY_FILE`, and `FDB_TLS_CA_FILE`.

## Restoring a Backup

You can start a restore by creating a restore object.
Here is an example restore, using the same account as the backup example above:

```yaml
apiVersion: apps.foundationdb.org/v1beta2
kind: FoundationDBRestore
metadata:
  name: sample-cluster
spec:
  destinationClusterName: sample-cluster
  blobStoreConfiguration:
    accountName: account@object-store.example:443
    backupName: sample-cluster
    bucketName: bucket=fdb-backups
```

This will tell the operator to run an `fdbrestore` command targeting the cluster `sample-cluster`. The cluster must be empty before this command can be run.
This will restore to the last restorable point in the backup you are using, and will restore the entire keyspace.

You can track the progress of the restore through the `fdbrestore status` command. The destination cluster will be locked until the restore completes.

### Backup agents for restore

When you start the restore against a new cluster, you have to ensure that backup agents are created by the `operator`.
If you don't want to backup the cluster directly you can add the `backupState: Stopped` under the `BackupSpec`, this will ensure that the backup agents are created but no backup is started:

```yaml
apiVersion: apps.foundationdb.org/v1beta2
kind: FoundationDBBackup
metadata:
  name: sample-cluster
spec:
  backupState: Stopped
  version: 7.1.26
  clusterName: sample-cluster
  blobStoreConfiguration:
    accountName: account@object-store.example:443
  podTemplateSpec:
    spec:
      volumes:
        - name: backup-credentials
          secret:
            secretName: backup-credentials
        - name: fdb-certs
          secret:
            secretName: fdb-certs
      containers:
        - name: foundationdb
          env:
            - name: FDB_BLOB_CREDENTIALS
              value: /var/backup-credentials/credentials
            - name: FDB_TLS_CERTIFICATE_FILE
              value: /var/fdb-certs/cert.pem
            - name: FDB_TLS_CA_FILE
              value: /var/fdb-certs/ca.pem
            - name: FDB_TLS_KEY_FILE
              value: /var/fdb-certs/key.pem
          volumeMounts:
            - name: fdb-certs
              mountPath: /var/fdb-certs
            - name: backup-credentials
              mountPath: /var/backup-credentials
```

### Debugging restores

In some cases it can happen that a restore is not properly started, in this case look at the operator logs for the according `FoundationDBRestore` resource.
Look for a message like `"Error from FDB command"` and verify the `stdout` of the command, if the destination cluster is not empty the restore command will return an error:

```text
No restore target version given, will use maximum restorable version from backup description.
Using target restore version 123
Backup Description
URL: blobstore://my-awesome-blobstore
Restorable: true
Partitioned logs: false
...
Restoring backup to version: 123
ERROR: Attempted to restore into a non-empty destination database
Fatal Error: Attempted to restore into a non-empty destination database
```

In this case check the content of the cluster e.g. by running:

```text
fdb> getrange "" \xff

Range limited to 25 keys
...
```

If the data is not required and should be removed run the following command:
**WARNING**: This will delete all user data from the cluster and cannot be undone.

```text
fdb> writemode on; clearrange "" \xff
```

After this check again that the cluster is empty:

```text
fdb> getrange "" \xff

Range limited to 25 keys
```

### Confirm restore is complete

You can run the `fdbrestore` command inside any FDB pod:

```text
$ fdbrestore status --dest_cluster_file ${FDB_CLUSTER_FILE}
Tag: default  UID: ...  State: completed  ...
```

The output will contain additional information about the restore, but the most interesting one is the `state`.
Once the restore is complete, the `state` changes to `completed`.

## Next

You can continue on to the [next section](technical_design.md) or go back to the [table of contents](index.md).
