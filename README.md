
# Automatic Secrets Management in ArgoCD with AWS Secrets Manager and Secrets Storage CSI Driver

# Disclaimer
This repo contains an opinionated demo and is NOT an official Red Hat documentation.

# Introduction

This repository's objective is to demonstrate the automatization of secrets management with ArgoCD using the AWS Secrets Manager and Secret Storage CSI Driver (SSCSI) in ARO.

## This repo is using the following tecnologies:
- ArgoCD
- Helm
- Secrets Storage CSI Driver (SSCSI)
- AWS Secrets Manager (ASM)


# Requirements
- ASM service will be used as the container of the secrets to be mounted in the pods.
- SSCSI will retrieve the secrets from ASM and mount as a Volume to the pods and as well create a K8s secret that will be passed as a enviremont variable in the Pods manifests.
- SSCSI is authenticated against the ASM using STS
- SSCSI by default mounts the retrived ASMv secrets retrived as a volume in the pods, nonetheless we have an additional requirement to mount as a K8s secret as well (details bellow). 

# SSCSI Flow
![Alt text](./pics/sscsi_flow1.png?raw=true "SSCSI ") 


# Procedure:

## Overview Steps

1. Clone the Git repository

2. Install GitOps

2. Install SSCSI in ROSA

3. Add Secret to AWS Secret Manager

4. Exercise: Developer creates a new application in GitOps:
    4. The developer creates a new Application in GitOps feeding the Git link of the application Helm chart  


## Detailed Procedure

1. Clone this Git repository
    - git clone https://github.com/CSA-RH/secrets_management-with-aws_secrets_manager-and_gitops.git
    - cd secrets_management-with-aws_secrets_manager-and_gitops

2. Install GitOps in your ROSA cluster
    - https://docs.openshift.com/gitops/1.12/installing_gitops/installing-openshift-gitops.html

3. Integrate AWS Secret Manager with ROSA
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
oc new-project ${NAMESPACE}
```

-  Label namespace to allow ArgoCD to manage the namespace
This is needed so that ARGOCD can manage K8s objects in the Namespace

```$bash
oc label namespace ${NAMESPACE} argocd.argoproj.io/managed-by=openshift-gitops
```

- Deploy the application in ArgoCD

```$bash
oc apply -f argocd/application_petclinic.yaml
```

## Notes

- Make sure to configure the Helm Chart values.yaml parameters: serviceaccount -> annotations, with the AWS Secret Manager secret Role ARN. Helm will add this Role ARN annotation to the ServiceAccount, so that the SA can be authenticated with the AWS Secret Manager service, by means of the STS token.  

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

    ```$bash
          jmesPath:
            - path: password
              objectAlias: db_password
    ```

    ![Alt text](./pics/acm_secret_format.png?raw=true "AWS Secret Manager Secret Format")

    1. Adding the jmesPath option the secret mount content will save only the value of the ACM saved secret.
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
