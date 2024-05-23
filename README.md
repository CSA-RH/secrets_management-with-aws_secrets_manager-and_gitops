
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
    - git clone https://github.com/CSA-RH/secrets_management-with-aws_secrets_manager-and_gitops.git

2. Install GitOps in your ROSA cluster
    - https://docs.openshift.com/gitops/1.12/installing_gitops/installing-openshift-gitops.html

3. Integrate AWS Secret Manager with ROSA - Install SSCSI/Create a secret in AWS Secret Manager/configure policies to give ROSA permissions to access ASM 
    1. Install procedure to install the community version:
        - https://access.redhat.com/documentation/en-us/red_hat_openshift_service_on_aws/4/html/tutorials/cloud-experts-aws-secret-manager

    2. The following additional steps are required if the SSCSI are required to create the K8s secrets:
        - add cluster role
            ```$bash
            oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:csi-secrets-store:secrets-store-csi-driver

            ??delete? Does not exist - dont seem to be needed??     #oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:k8s-secrets-store-csi:secrets-store-csi-driver
            ```

        - create custom SCC
            ?? Not sure if is neeeded -> test      To allow usaging CSI volume and adding the capability "SETGID" (this capability as it is required by the MySQL DDBB Image).

            ```$bash
            oc apply -f clusterprimer/scc.yaml
            ```

        - add role
            ?? Not sure if is neeeded -> test
            ```$bash
            oc apply -f clusterprimer/ClusterRole_scc.yaml 
            ```

# Demo: Developer creates a new application in GitOps:
## Export Enviremont variables to your local laptop

```$bash
export NAMESPACE=petclinic3

```

- Make sure to configure the Helm Chart values.yaml parameters: serviceaccount -> annotations, with the AWS Secret Manager secret Role ARN. Helm will add this Role ARN annotation to the ServiceAccount, so that the SA can be authenticated with the AWS Secret Manager service, by means of the STS token.  

- Create namespace and add label to allow GitOps to manage the namespace

```$bash
oc new-project ${NAMESPACE}
```

```$bash
This is needed so that ARGOCD can manage K8s objects in the Namespace
oc label namespace ${NAMESPACE} argocd.argoproj.io/managed-by=openshift-gitops
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
    repoURL: https://github.com/CSA-RH/secrets_management-with-aws_secrets_manager-and_gitops.git
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
