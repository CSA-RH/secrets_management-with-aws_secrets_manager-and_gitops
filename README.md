
# Automatic Secrets Management in ArgoCD with AWS Secrets Manager and Secrets Storage CSI Driver

# Disclaimer
This repo contains an opinionated demo and is NOT an official Red Hat documentation.

# Introduction

This repository's objective is to demonstrate the automatization of secrets management with ArgoCD using the AWS Secrets Manager and Secret Storage CSI Driver (SSCSI) in ROSA.

## This repo is using the following tecnologies:
- ROSA HCP or Classic Cluster
- ArgoCD
- Helm
- Secrets Storage CSI Driver (SSCSI)
- AWS Secrets Manager (ASM)


# Requirements
- ASM service will be used as the Vault container of the secrets.
- SSCSI will retrieve the secrets from ASM and mount as a Volume to the pods and as well create a K8s secret that will be passed as a enviremont variable in the Pods manifests.
- OOTB SSCSI mounts the secrets retrived from ASM, as a volume mounted in the Pods.nonetheless we have an additional requirement to mount as a K8s Secret as well.

# SSCSI Flow
![Alt text](./pics/sscsi_flow1.png?raw=true "SSCSI ") 


# Procedure:

## Overview Steps

1. Clone the Git repository

2. Install GitOps

2. Install SSCSI in ROSA

3. Add Secret to AWS Secret Manager

4. Demo: Deploy Workload with ARGOCD  


## Detailed Procedure

1. Clone this Git repository
    - git clone https://github.com/CSA-RH/secrets_management-with-aws_secrets_manager-and_gitops.git
    - cd secrets_management-with-aws_secrets_manager-and_gitops

2. Install GitOps in your ROSA cluster
    - https://docs.openshift.com/gitops/1.12/installing_gitops/installing-openshift-gitops.html

3. Integrate AWS Secret Manager with ROSA. Red Hat has an SSCSI Operator wich is Tech Preview (https://docs.openshift.com/container-platform/4.17/storage/container_storage_interface/persistent-storage-csi-secrets-store.html), though for this demo Im installing SSCSI Community Helm chart - Note the Community software is not supported by Red Hat.
    1. Procedure to install the community version and create a  secret in AWS Secret Manager and configure policies to give ROSA permissions to access ASM:
        - https://access.redhat.com/documentation/en-us/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-aws-secret-manager

    2. The following additional step is required if the SSCSI has to create the K8s secret:
        - add cluster role
            ```$bash
            oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:csi-secrets-store:secrets-store-csi-driver

    3. The following custom SCC is required by the MySQL image used that needs the SETGID capability. 
        - create custom SCC

            ```$bash
            oc apply -f clusterprimer/scc.yaml
            ```

# Demo: Deploy Workload with ARGOCD

- Create namespace

    ```$bash
    export NAMESPACE=petclinic3
    ```

    ```$bash
    oc new-project ${NAMESPACE}
    ```

-  Label namespace to allow ArgoCD to manage the namespace

    ```$bash
    oc label namespace ${NAMESPACE} argocd.argoproj.io/managed-by=openshift-gitops
    ```

- Deploy the application in ArgoCD

    ```$bash
    cat <<EOF | oc apply -f -
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: $NAMESPACE
      namespace: openshift-gitops
    spec:
      destination:
        namespace: $NAMESPACE
        server: https://kubernetes.default.svc
      project: default
      source:
        helm:
          valueFiles:
          - values.yaml
        path:  helm/pet-clinic/
        repoURL: https://github.com/CSA-RH/secrets_management-with-aws_secrets_manager-and_gitops.git
        targetRevision: HEAD
    EOF
    ```

## Notes

- Make sure to configure the Helm Chart values.yaml parameters: serviceaccount -> annotations, with the AWS Secret Manager secret Role ARN. Helm will add this Role ARN annotation to the ServiceAccount, so that the ServiceAccount can be authenticated with the AWS Secret Manager service.  

- How can I instruct to create a K8s Secret in addition to mount the secret retried as a volume in the POD? By configuring additional parameters in the SecretProviderClass. Example:

    1. Configure the SecretProviderClass
        ```$bash
          secretObjects:
          - data:
            - objectName: db_password
              key: password
            secretName: {{ .Values.secretprovider.secretName }}
            type: Opaque
        ```

    2. Configure the POD manifest to consume the K8s secret
        ```$bash
            - name: MYSQL_PASSWORD          
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.secretprovider.secretName }}
                  key: password
        ```

- The AWS Secret Manager saves the secret as a tuple (key:value). This means that the secret retrived from ACM and mounted in a POD volume, will have this format as well, i.e. key:value, exemple password:petclinic. If the application that has to consume this secret requires only the value this can be configured in the SecretProviderClass as follows:

    1. Configure the SecretProviderClass with the parameter jmesPath.

        ```$bash
              jmesPath:
                - path: password
                  objectAlias: db_password
        ```

    2. Example of ASM were it shows the Secret tuple 
  
        ![Alt text](./pics/acm_secret_format.png?raw=true "AWS Secret Manager Secret Format")

    3. Adding the jmesPath option the POD secret mount content will only contain the value of the ACM secret:

        ```$bash
        oc exec mysql-deployment-ffc755464-vpg5f -- cat /mnt/secrets-store/db_password

        petclinic
        ```

# To Delete the Application

```$bash
oc -n openshift-gitops delete Application $NAMESPACE
```

```$bash
oc delete project $NAMESPACE
```
