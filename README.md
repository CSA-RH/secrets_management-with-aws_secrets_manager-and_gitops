
# Automatic Secrets Management in ArgoCD with AWS Secrets Manager and Secrets Storage CSI Driver

# Disclaimer
This repo contains an opinionated demo and is NOT an official Red Hat recommendation.

# Introduction

This repository's objective is to demonstrate the automatization of secrets management with ArgoCD using the AWS Secrets Manager and Secret Storage CSI Driver (SSCSI) in ARO.

## This repo is using the following tecnologies:
- GitOps/ArgoCD
- Helm
- Secrets Storage CSI Driver (SSCSI)
- AWS Secrets Manager (ASM)


# Requirements
- ASM service will be used as the container of the secrets to be mounted in the pods.
- SSCSI will retrieve the secrets from ASM and mount as a Volume to the pods and as well create a K8s secret that will be passed as a enviremont variable in the Pods manifests.
- SSCSI is authenticated against the ASM using STS
- SSCSI by default mounts the retrived ASMv secrets retrived as a volume in the pods, nonetheless we have an additional requirement to mount as a K8s secret as well (details bellow). 


# Procedure:

## Overview Steps

1. Clone the Git repository

2. Install GitOps

2. Install SSCSI in ROSA

3. Add Secret to AWS Secret Manager

4. Exercise: Developer creates a new application in GitOps:
    4. The developer creates a new Application in GitOps feeding the Git link of the application Helm chart  



## Detailed Procedure

1. Clone the Git repository into your laptop
?????git clone https://github.com/CSA-RH/helm-gitops-sscsi_secrets.git

2. Install GitOps in your ROSA cluster
    - https://docs.openshift.com/gitops/1.12/installing_gitops/installing-openshift-gitops.html

3. Integrate AWS Secret Manager with ROSA - Install SSCSI/Create a secret in AWS Secret Manager/configure policies to give ROSA permissions to access ASM 
    1. Install procedure to install the community version:
        - https://access.redhat.com/documentation/en-us/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-aws-secret-manager

??    2. The following additional steps are required if the SSCSI are required to create an K8s secret:
        - add cluster role

            ```$bash
            oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:k8s-secrets-store-csi:secrets-store-csi-driver
            ```

        - create custom SCC
            To allow usaging CSI volume and adding the capability "SETGID" (this capability as it is required by the MySQL DDBB Image).

            ```$bash
            oc apply -f clusterprimer/scc.yaml
            ```

       - add role

            ```$bash
            oc apply -f clusterprimer/ClusterRole_scc.yaml 
            ```

# Demo: Developer creates a new application in GitOps:
## Export Enviremont variables to your local laptop

```$bash
export ASM_RESOURCE_GROUP=lmartinh-rgb6
export ASM_NAME=lmartinh-vault-1
export ASM_LOCATION=eastus
export NAMESPACE=petclinic3
export SECRET_NAME=demosecret4
export ASM_VALUE=petclinic

#Service Principal Key to access the Azure Key Vault 
export ASM_SECRET_NAME=secrets-store-creds
```


- Create namespace and add label to allow GitOps to manage the namespace

```$bash
oc new-project ${NAMESPACE}
```

```$bash
oc label namespace ${NAMESPACE} argocd.argoproj.io/managed-by=openshift-gitops
```

- Give authorization to the namespace to call the bitnami API Group 
> ####Moved to Git#####
#oc adm policy add-role-to-user admin-sealedsecret system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n ${NAMESPACE}
```

- Before creating the application it is necessary to make a commit and push to the forked repository. 

```$bash
git add *
git commit -m "argocd sealed secrets"
git push
```

- Deploy the application in GitOps

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
    repoURL: https://github.com/CSA-RH/helm-gitops-sscsi_secrets.git
    targetRevision: HEAD
EOF
```

# To Delete the Application

```$bash
oc -n openshift-gitops delete Application $NAMESPACE
```

```$bash
oc delete project $NAMESPACE
```
