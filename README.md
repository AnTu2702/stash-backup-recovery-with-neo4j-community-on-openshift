Repository refers to the following video on youtube:
https://www.youtube.com/watch?v=SJafvdWHAhg

- appscode/stash for Kubernetes applied restic backup to your cluster:

  https://appscode.com/products/stash/0.7.0/concepts/

- offline backup vs online backup

  https://appscode.com/products/stash/0.7.0/guides/offline_backup/

  When a Restic is created with spec.type: offline, stash operator adds an init-container instead of sidecar container to target workload pods. The init-container takes backup once. If the backup is successfully completed, then it creates a job to perform restic check and exits. The app container starts only after the init-container exits without any error. This ensures that the app container is not running while taking backup. Stash operator also creates a cron-job that deletes the workload pods according to the spec.schedule. Thus the workload pods get restarted periodically and allow the init-container to take backup.

- setup on openshift
  
  - installation

    **Attention:** appscode/stash will run properly only an openshift 3.9 and greater
    https: //github.com/appscode/stash/tree/openshift

    procedure:

    ```
    oc project kube-system

    curl -fsSL https://raw.githubusercontent.com/appscode/stash/0.7.0/hack/deploy/stash.sh | bash
    ```
    **remark:** *proxy/noproxy env vars may have to be set to the operator deployment*

    the installation via helm is also possible see here: https://appscode.com/products/stash/0.7.0/setup/install/

  - credentials

    for usage of appscode/stash as well as for making it use an s3-bucket for backup repository some openshift secret has to be created in the application namespace:

    ```
    #!/bin/bash

    # Your credentials
    echo -n 'changeit' > RESTIC_PASSWORD
    echo -n 'AKIA...............' > AWS_ACCESS_KEY_ID
    echo -n 'RIDM...........................' > AWS_SECRET_ACCESS_KEY

    # The secret is required in your namespace...
    oc project <your-application-namespace>

    # So create it there...
    oc create secret generic s3-secret \
        --from-file=./RESTIC_PASSWORD \
        --from-file=./AWS_ACCESS_KEY_ID \
        --from-file=./AWS_SECRET_ACCESS_KEY

    # Do clean up
    rm -f RESTIC_PASSWORD
    rm -f AWS_ACCESS_KEY_ID
    rm -f AWS_SECRET_ACCESS_KEY
    ```

    Remark: The AWS access credentials should not be required in case of instance policy usage!

  - known issues

    + In order to have stash/restic working properly on the openshift clusteer, the serviceaccount "default" requires some higher policy. For the moment assign admin policy to it... (should be debugged and restricted to correct policy only...)
    + Keep an eye on the s3 endpoint configuration. For connecting to s3 buckets in eu-central-1, the correct endpoint is: 's3-eu-central-1.amazonaws.com'
    + Prevent to use mountpoints as backup folders. Restic will try to set/overwrite filesystem attributes after recovery. This will fail on mountpoints with se-linux error. Rather use subdirectories to be backed up.
    + Stash on Openshift will use the deployment number as basis for the backup repository. A new deployment will create a new repository!
  - open questions

    + How can stash be adviced to take an existing repository in an s3-bucket (without overwriting existing backups e.g. from previous cluster/deployment seqno?)
    + Does Stash require RWX storage?

- usage scenario

  ```
  backup-example.yaml:

  apiVersion: stash.appscode.com/v1alpha1
  kind: Restic
  metadata:
    name: neo4j
    namespace: myproject
  spec:
    selector:
      matchLabels:
        app: neo4j
    type: offline
    fileGroups:
    - path: /data/dbms
      retentionPolicyName: 'keep-last-5'
    - path: /data/databases
      retentionPolicyName: 'keep-last-5'
    backend:
      s3:
        endpoint: 's3-eu-central-1.amazonaws.com'
        bucket: your-s3-backup-repo-bucket-name
        prefix: your-repository-folder
      storageSecretName: s3-secret
    schedule: '@every 2m'
    volumeMounts:
    - mountPath: /data
      name: neo4j-data
    retentionPolicies:
    - name: 'keep-last-5'
      keepLast: 5
      prune: true
  ```
  ```oc create -f ./backup-example.yaml```

  ```
  restore-example.yaml:

  apiVersion: stash.appscode.com/v1alpha1
    kind: Recovery
    metadata:
      name: neo4j
      namespace: myproject
    spec:
      backend:
        s3:
          bucket: neo4j-stash-eval-s3-bucket
          endpoint: s3-eu-central-1.amazonaws.com
          prefix: neo4j-backup-repository
        storageSecretName: s3-secret
      paths:
      - /data/databases
      - /data/dbms
      recoveredVolumes:
      - mountPath: /data
        persistentVolumeClaim:
          claimName: neo4j-data
      repository:
        name: replicationcontroller.neo4j-4
        namespace: myproject
      snapshot: replicationcontroller.neo4j-4-9264a124
      type: offline
      workload:
        kind: ReplicationController
        name: neo4j-4
  ```
  ```oc create -f ./restore-example.yaml```

- crd integration:

  restic:
  ```
    $> oc get restic
      NAME     AGE
      neo4j    4m
    $> oc describe restic
    Name:         neo4j
    Namespace:    myproject
    Labels:       <none>
    # Please edit the object below. Lines beginning with a '#' will be ignored,
    Annotations:  <none>
    API Version:  stash.appscode.com/v1alpha1
    Kind:         Restic
    Metadata:
      Cluster Name:
      Creation Timestamp:  2018-08-09T09:19:48Z
      Resource Version:    9906
      Self Link:           /apis/stash.appscode.com/v1alpha1/namespaces/myproject/restics/neo4j
      UID:                 5ccc725d-9bb5-11e8-9135-2a5a0ecf7d79
    Spec:
      Backend:
        S 3:
          Bucket:             neo4j-stash-eval-s3-bucket
          Endpoint:           s3-eu-central-1.amazonaws.com
          Prefix:             neo4j-backup-repository
        Storage Secret Name:  s3-secret
      File Groups:
        Path:                   /data
        Retention Policy Name:  keep-last-5
      Retention Policies:
        Keep Last:  5
        Name:       keep-last-5
        Prune:      true
      Schedule:     @every 2m
      Selector:
        Match Labels:
          App:  neo4j
      Type:     offline
      Volume Mounts:
        Mount Path:  /data
        Name:        neo4j-data
    Events:
      Type    Reason           Age   From         Message
      ----    ------           ----  ----         -------
      Normal  SuccessfulCheck  2m    stash-check  Check successful for pod: neo4j-4
      Normal  SuccessfulCheck  30s   stash-check  Check successful for pod: neo4j-4
  ```

  repository:
  ```
  $> oc get repository
  NAME                           AGE
  replicationcontroller.neo4j-4  4m
  ```


  snapshot:
  ```
  $> oc get snapshot
  NAME                                    AGE
  replicationcontroller.neo4j-4-ff4bfa92  9m
  replicationcontroller.neo4j-4-bd239dfe  7m
  replicationcontroller.neo4j-4-41ff1c38  5m
  replicationcontroller.neo4j-4-70dfc73e  3m
  replicationcontroller.neo4j-4-9a4827e9  1m
  ```


  recovery:
  ```
  oc get recovery

  Name:         neo4j
  Namespace:    myproject
  Labels:       <none>
  Annotations:  <none>
  API Version:  stash.appscode.com/v1alpha1
  Kind:         Recovery
  Metadata:
    Cluster Name:
    Creation Timestamp:  2018-08-09T12:04:10Z
    Generation:          0
    Resource Version:    13730
    Self Link:           /apis/stash.appscode.com/v1alpha1/namespaces/myproject/recoveries/neo4j
    UID:                 53572d63-9bcc-11e8-9135-2a5a0ecf7d79
  Spec:
    Backend:
      S 3:
        Bucket:             neo4j-stash-eval-s3-bucket
        Endpoint:           s3-eu-central-1.amazonaws.com
        Prefix:             neo4j-backup-repository
      Storage Secret Name:  s3-secret
    Paths:
      /data
    Recovered Volumes:
      Mount Path:  /data
      Persistent Volume Claim:
        Claim Name:  neo4j-data
    Repository:
      Name:       replicationcontroller.neo4j-4
      Namespace:  myproject
    Snapshot:     replicationcontroller.neo4j-4-9a4827e9
    Type:         offline
    Workload:
      Kind:  ReplicationController
      Name:  neo4j-4
  Status:
    Phase:  Running
  Events:
    Type    Reason              Age   From              Message
    ----    ------              ----  ----              -------
    Normal  RecoveryJobCreated  1m    stash-controller  Recovery job created: stash-recovery-neo4j
  ```
