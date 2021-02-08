---
title:  Restore ICD Database from PIT Backup
description:
updated: 2021-02-09
version:1.0
---

# ICD recover from a Point-In-Time (PIT) backup

??? quote "Something missing?"
    [Raise an issue][github-issue] in our issue tracker.

## Overview

<!-- Add contextual or background information. -->
IBM® Cloud Databases for PostgreSQL offers Point-In-Time Recovery (PITR) for any time in the last 7 days. The deployment performs continuous incremental backups and can replay transactions to bring a new deployment that is restored from a backup to any point in that 7-day window you need

Backups are restored to a new deployment. After the new deployment finishes provisioning, your data in the backup file is restored into the new deployment.

<!-- ## Prerequisites -->

<!-- List prerequisites (if any). -->

## ICD recovery from a Point-In-Time (PIT) backup

<!-- If your runbook involves task-steps, you can organize your document by
    highlighting the major steps (Step 1, Step 2), and then including sub-steps
    within each section.
-->
To recover an ICD from a Point-In-Time (PIT) backup the following steps are required:

1.  login IBM cloud via cli:
    ```text
    ibmcloud login -r us-south -g Default -a https://cloud.ibm.com --apikey <YOUR KEY>
    ```

2.  list all the ICD deployments:
    ```text
    ibmcloud cdb ls
    ```

3.  list all tasks/operations on a specific deployment:
    ```text
    ibmcloud cdb tasks deployment-name <e.g. keyprotect-kpp-stage-icd>
    ```

4.  get the deployment ID/CRN:
    ```text
    ibmcloud cdb about deployment-name 
    ```
    example
    ```
    crn:v1:bluemix:public:databases-for-postgresql:us-south:a/c0a097215c15ee1571f730811d77b014:92e55dea-b60e-4797-85b0-8397927b4a77::
    ```

5.  get the earliest PIT backup available:
    ```text
    ibmcloud cdb postgresql earliest-pitr-timestamp deployment-name
    ```

6.  create a new ICD deployment from a PIT backup:
    ```
    ibmcloud resource service-instance-create <SERVICE_INSTANCE_NAME> <service-id> <region> -p '{"point_in_time_recovery_deployment_id":"DEPLOYMENT_ID/CRN", "point_in_time_recovery_time":"TIMESTAMP"}'
    ```

    a. restore from the latest PIT backup (in us-east):
    
    example:
    ```
    ibmcloud resource service-instance-create KPP-RESTORE-PITR-TEST databases-for-postgresql standard us-east -p '{"point_in_time_recovery_time":"","point_in_time_recovery_deployment_id":"crn:v1:bluemix:public:databases-for-postgresql:us-south:a/c0a097215c15ee1571f730811d77b014:92e55dea-b60e-4797-85b0-8397927b4a77::"}'
    ```

    b. restore from a backup created at some point in time (in us-east), e.g. 2020-07-25T22:13:27Z:
    
    example: 
    ```
    ibmcloud resource service-instance-create KPP-RESTORE-PITR-TEST databases-for-postgresql standard us-east -p '{"point_in_time_recovery_time":"","point_in_time_recovery_deployment_id":"crn:v1:bluemix:public:databases-for-postgresql:us-south:a/c0a097215c15ee1571f730811d77b014:92e55dea-b60e-4797-85b0-8397927b4a77::", "point_in_time_recovery_time":"2020-07-25T22:13:27Z"}'
    ```

7.  check ICD restore status with this bash script:
    ```
    continue="true"
    while [ "$continue" == "true" ]
    do
        date 
        if ibmcloud cdb ls | grep KPP-RESTORE-PITR-TEST | grep -q provisioning;
        then 
            echo ".... still provisioning ..."
            sleep 600
        else
            echo ".... Restore DONE! ..." 
            continue="false"
        fi
    done
    ```


8.  Create a "Service credential" for the newly restored ICD instance (KPP-RESTORE-PITR-TEST) in the cloud.ibm.com UI.

    a. login https://cloud.ibm.com/resources

    b. go to "Resource list" --> "Services" --> "the newly restored ICD instance (KPP-RESTORE-PITR-TEST)"

    c. on the left panel, click on "Service credentials"

    d. Click on the "New credential", specify a name on the popup and click "Add"

    e. all the info needed for the next step are in this new "Service credentials" instance


9.  update Key Protect's configs to point to the newly restored ICD deployment:

    A) update ICD secrets in vault for the targeted env, example: kpp

    - generic/crn/v1/bluemix/public/kms/us-south/kpp/databases/icd-postgresql/host
    - generic/crn/v1/bluemix/public/kms/us-south/kpp/databases/icd-postgresql/kpservice_cert
    - generic/crn/v1/bluemix/public/kms/us-south/kpp/databases/icd-postgresql/kpservice_password
    - generic/crn/v1/bluemix/public/kms/us-south/kpp/databases/icd-postgresql/port
    - generic/crn/v1/bluemix/public/kms/us-south/kpp/databases/icd-postgresql/user

    B) deploy or redeploy the cluster for pgbouncer to pick up the new ICD settings.  
    
    **Note:** The deploy automation now also runs secretsMergehttps://wcp-kms-team-jenkins.swg-devops.com/job/OPS/job/Deploy/

10.  verify KP is working, pointing to the newly created ICD instance and KP is functional (via kp-regress)


**Note:** ICD PITR backups are created every 30 minutes or 16 MB of data written.

