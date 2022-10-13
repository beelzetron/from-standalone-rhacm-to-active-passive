# How to move from standalone RHACM to Active/Passive Setup

Recently one of our Customers asked how to move from a standalone RHACM setup to an Active/Passive Hub one given that the [Hub Backup and Restore feature went GA on Red Hat Advanced Cluster Management 2.5](https://www.redhat.com/en/blog/red-hat-brings-greater-simplicity-and-flexibility-kubernetes-management-latest-version-red-hat-advanced-cluster-management-kubernetes).

The Customer had a **3 data-centers setup**, each one with its own RHACM Cluster Hub responsible of managing the local OCP clusters. They planned their setup with disaster recovery in mind so there were no object name collisions. The initial DR plan was to manually import OpenShift clusters of the failed Cluster Hub into one of the other two.

![Standalone RHACM Hubs](images/rhacm-consolidation-standalone-hubs.png)

All the policies were synchronized on all the Cluster Hubs via GitOps, while this approach permitted a sort of hot standby capability - importing clusters would've been enough to start applying policies on them - it caused a **lot of policies with unknown state** since the clusters were they were supposed to be applied weren't present.

Another problem was the **loss of cluster creation data in case of Cluster Hub failure**: our Customer leveraged RHACM to create OpenShift clusters on VMware infrastructure running on their 3 data-centers, this data would not be automatically imported into another Cluster Hub.

Given all of that, Customer was really keen to adopt the new Business Continuity model offered by RHACM 2.5 Backup and Restore feature. We'll not discuss the setup of the feature here - you can find a great explanation [here](https://cloud.redhat.com/blog/backup-and-restore-hub-clusters-with-red-hat-advanced-cluster-management-for-kubernetes) - we'll focus instead on the procedure adopted to move from 3 standalone RHACM Cluster Hubs into one Active Cluster Hub and two Passive Cluster Hubs with no data loss.

RHACM Backup and Restore feature was leveraged by [Red Hat Consulting](https://www.redhat.com/en/services/consulting) to move the managed clusters from the *soon-to-be* Passive Hubs into the designated Primary Hub by following this procedure:

1. Hub Backup was configured on all the RHACM cluster hubs, each Hub initially pointed to a dedicated prefix on the object storage bucket to have a valid backup for each of the hubs separately.

   Hub1 example `DataProtectionApplication`:

   ```yaml
   apiVersion: oadp.openshift.io/v1alpha1
   kind: DataProtectionApplication
   metadata:
     name: dpa-hub1
   spec:
     configuration:
       velero:
         defaultPlugins:
         - openshift
         - aws
       restic:
         enable: false
     backupLocations:
       - name: default
         velero:
           provider: aws
           default: true
           objectStorage:
             bucket: my-bucket
             prefix: hub1 # <<<< Set a dedicated prefix for each of the hubs
           config:
             region: us-east-1
             profile: "default"
           credential:
             name: cloud-credentials
             key: cloud
     snapshotLocations:
       - name: default
         velero:
           provider: aws
           config:
             region: us-west-2
             profile: "default"
   ```

2. 
