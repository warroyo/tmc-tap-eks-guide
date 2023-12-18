# TAP on EKS using TMC 

This repo is an example of how to quickly get up and running in EKS with TAP using the Tanzu CLI for TMC. The goal of this repo is to simplify all of the pieces involved in standing up TAP down to some CLI commands. The goal of this is not to entirely script the process, but rather outline and document all of the individual commands needed to set up TAP in an opinionated way on EKS using TMC. This repo does not create an iterate cluster at this time. the current focus of this repo is on the outer loop.

## What this does

This repo will create a fully working multicluster TAP install using TMC. This uses native AWS auth with IRSA as well as ECR as the repo. Additionally this sets up gitops with TMC managed flux that enables installing extra software in the clusters and managing TAP workloads through gitops OOTB.

## Tools

* Tanzu CLI
  * TMC plugin
  * Apps Plugin
* yq
* ytt
* AWS cli
* eksctl

## Usage

Clone/Fork this repo and follow the steps below. 

## Populate the values file

We will try and capture as much as we possibly can in the `values` to reduce the need to manually rename things. However there are still some things that need to be manually renamed, those will be documented in the steps below.

The values file can be found in `tanzu-cli/values/values.yml`. Update all of the necessary fields to match your environment.

## Setup the infra 

First we need to create some clusters and setup our intial gitops repo.

### Create the cluster group

This creates the intiial cluster group that all of the tap clusters will be part of and is the base for our gitops setup.

```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cluster-group/cg-template.yml > generated/cg.yaml
tanzu tmc clustergroup create -f generated/cg.yaml
```


### Enable helm and flux

The below commands enable flux at the cluster group level and install the source, helm, and kustomize controllers. These will be installed automatically on all clusters in this cluster group.

```
tanzu tmc continuousdelivery enable -g tap-mc -s clustergroup
tanzu tmc helm enable -g tap-mc -s clustergroup
```

### Create cluster group secrets

We can use the TMC cluster group secrets feature to push out opaque or registry credentials to all of out k8s clusters. For this use case we will use it to add our github pat as a secret to be used later in the TAP install. 

1. generate a PAT in github using the legacy token method. Give the pat access to all repo privileges.
2. copy the `tanzu-cli/values/sensitive-values-template.yml` to `tanzu-cli/values/sensitive-values.yml` and add your newly created PAT and username to it. 
3. create and export the secret using the below commands
```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/github-pat-template.yml > generated/pat-secret.yaml
tanzu tmc secret create -f generated/pat-secret.yaml -s clustergroup

ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/github-pat-export-template.yml > generated/pat-export.yaml
tanzu tmc secret export create -f generated/pat-export.yaml -s clustergroup
```

### Create the clusters

This needs to be done for 3 clusters, each with a different profile
* build
* run
* view
  
```
export $PROFILE=<profile-name>
ytt --data-values-file tanzu-cli/values --data-value profile=$PROFILE -f tanzu-cli/clusters/cluster-template.yml > generated/$PROFILE-cluster.yaml

ytt --data-values-file tanzu-cli/values --data-value profile=$PROFILE -f tanzu-cli/clusters/nodepool-template.yml > generated/$PROFILE-np.yaml

tanzu tmc ekscluster create -f $PROFILE-cluster.yaml
tanzu tmc ekscluster nodepool create -f $PROFILE-np.yaml                                                                 
```

### asscoiate the OIDC url with a provider


This sets up the OIDC provdier in aws so that we can use IRSA. Run this for all three clusters

```
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve
```

### Create ECR repos 

[official docs](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/install-aws-resources.html#create-the-workload-container-repositories-5)

ECR is a bit different than other registries, for ECR everything needs to be pre-created so we need to create repos for both the build service images and the workloads. I this example we are creating one for `tanzu-java-web-app-dev` but if you add more workloads you will need to create ECR repos for those as well. 

```
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export AWS_REGION=us-west-2

aws ecr create-repository --repository-name tanzu-application-platform/tanzu-java-web-app-dev --region $AWS_REGION
aws ecr create-repository --repository-name tap-build-service --region $AWS_REGION

```


### Create the AWS roles/policies for ECR

[official docs](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/install-aws-resources.html#create-iam-roles-6)

This sets up two roles with some inline policy and a trust policy. This is required for using IRSA with ECR and allows the build service to pull and push images via the role. These should be applied to the build cluster.

export the required vars
```
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export AWS_REGION=us-west-2
export EKS_CLUSTER_NAME=$(cat tanzu-cli/values/values.yml| yq .clusters.build.name)
export OIDCPROVIDER=$(aws eks describe-cluster --name $EKS_CLUSTER_NAME --region $AWS_REGION --output json | jq '.cluster.identity.oidc.issuer' | tr -d '"' | sed 's/https:\/\///')
```

create the build service trust policy and role

```
cat << EOF > generated/build-service-trust-policy-$EKS_CLUSTER_NAME.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDCPROVIDER}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${OIDCPROVIDER}:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "${OIDCPROVIDER}:sub": [
                        "system:serviceaccount:kpack:controller",
                        "system:serviceaccount:build-service:dependency-updater-controller-serviceaccount"
                    ]
                }
            }
        }
    ]
}
EOF

aws iam create-role --role-name tap-build-service-$EKS_CLUSTER_NAME --assume-role-policy-document file://generated/build-service-trust-policy-$EKS_CLUSTER_NAME.json

```


create the build service permissions policy

```
cat << EOF > generated/build-service-policy-$EKS_CLUSTER_NAME.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ecr:DescribeRegistry",
                "ecr:GetAuthorizationToken",
                "ecr:GetRegistryPolicy",
                "ecr:PutRegistryPolicy",
                "ecr:PutReplicationConfiguration",
                "ecr:DeleteRegistryPolicy"
            ],
            "Resource": "*",
            "Effect": "Allow",
            "Sid": "TAPEcrBuildServiceGlobal"
        },
        {
            "Action": [
                "ecr:DescribeImages",
                "ecr:ListImages",
                "ecr:BatchCheckLayerAvailability",
                "ecr:BatchGetImage",
                "ecr:BatchGetRepositoryScanningConfiguration",
                "ecr:DescribeImageReplicationStatus",
                "ecr:DescribeImageScanFindings",
                "ecr:DescribeRepositories",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetLifecyclePolicy",
                "ecr:GetLifecyclePolicyPreview",
                "ecr:GetRegistryScanningConfiguration",
                "ecr:GetRepositoryPolicy",
                "ecr:ListTagsForResource",
                "ecr:TagResource",
                "ecr:UntagResource",
                "ecr:BatchDeleteImage",
                "ecr:BatchImportUpstreamImage",
                "ecr:CompleteLayerUpload",
                "ecr:CreatePullThroughCacheRule",
                "ecr:CreateRepository",
                "ecr:DeleteLifecyclePolicy",
                "ecr:DeletePullThroughCacheRule",
                "ecr:DeleteRepository",
                "ecr:InitiateLayerUpload",
                "ecr:PutImage",
                "ecr:PutImageScanningConfiguration",
                "ecr:PutImageTagMutability",
                "ecr:PutLifecyclePolicy",
                "ecr:PutRegistryScanningConfiguration",
                "ecr:ReplicateImage",
                "ecr:StartImageScan",
                "ecr:StartLifecyclePolicyPreview",
                "ecr:UploadLayerPart",
                "ecr:DeleteRepositoryPolicy",
                "ecr:SetRepositoryPolicy"
            ],
            "Resource": [
                "arn:aws:ecr:${AWS_REGION}:${AWS_ACCOUNT_ID}:repository/full-deps-package-repo",
                "arn:aws:ecr:${AWS_REGION}:${AWS_ACCOUNT_ID}:repository/tap-build-service",
                "arn:aws:ecr:${AWS_REGION}:${AWS_ACCOUNT_ID}:repository/tap-images"
            ],
            "Effect": "Allow",
            "Sid": "TAPEcrBuildServiceScoped"
        }
    ]
}
EOF

aws iam put-role-policy --role-name tap-build-service-$EKS_CLUSTER_NAME --policy-name tapBuildServicePolicy --policy-document file://generated/build-service-policy-$EKS_CLUSTER_NAME.json
```

create the workload trust policy specific to the cluster's OIDC endpoint 

```
cat << EOF > generated/workload-trust-policy-$EKS_CLUSTER_NAME.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDCPROVIDER}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${OIDCPROVIDER}:sub": "system:serviceaccount:dev:default",
                    "${OIDCPROVIDER}:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
EOF

aws iam create-role --role-name tap-workload-$EKS_CLUSTER_NAME --assume-role-policy-document file://generated/workload-trust-policy-$EKS_CLUSTER_NAME.json
```

create the workload permissions policy specific to the account.

```
cat << EOF > generated/workload-policy-$EKS_CLUSTER_NAME.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ecr:DescribeRegistry",
                "ecr:GetAuthorizationToken",
                "ecr:GetRegistryPolicy",
                "ecr:PutRegistryPolicy",
                "ecr:PutReplicationConfiguration",
                "ecr:DeleteRegistryPolicy"
            ],
            "Resource": "*",
            "Effect": "Allow",
            "Sid": "TAPEcrWorkloadGlobal"
        },
        {
            "Action": [
                "ecr:DescribeImages",
                "ecr:ListImages",
                "ecr:BatchCheckLayerAvailability",
                "ecr:BatchGetImage",
                "ecr:BatchGetRepositoryScanningConfiguration",
                "ecr:DescribeImageReplicationStatus",
                "ecr:DescribeImageScanFindings",
                "ecr:DescribeRepositories",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetLifecyclePolicy",
                "ecr:GetLifecyclePolicyPreview",
                "ecr:GetRegistryScanningConfiguration",
                "ecr:GetRepositoryPolicy",
                "ecr:ListTagsForResource",
                "ecr:TagResource",
                "ecr:UntagResource",
                "ecr:BatchDeleteImage",
                "ecr:BatchImportUpstreamImage",
                "ecr:CompleteLayerUpload",
                "ecr:CreatePullThroughCacheRule",
                "ecr:CreateRepository",
                "ecr:DeleteLifecyclePolicy",
                "ecr:DeletePullThroughCacheRule",
                "ecr:DeleteRepository",
                "ecr:InitiateLayerUpload",
                "ecr:PutImage",
                "ecr:PutImageScanningConfiguration",
                "ecr:PutImageTagMutability",
                "ecr:PutLifecyclePolicy",
                "ecr:PutRegistryScanningConfiguration",
                "ecr:ReplicateImage",
                "ecr:StartImageScan",
                "ecr:StartLifecyclePolicyPreview",
                "ecr:UploadLayerPart",
                "ecr:DeleteRepositoryPolicy",
                "ecr:SetRepositoryPolicy"
            ],
            "Resource": [
                "arn:aws:ecr:${AWS_REGION}:${AWS_ACCOUNT_ID}:repository/full-deps-package-repo",
                "arn:aws:ecr:${AWS_REGION}:${AWS_ACCOUNT_ID}:repository/tanzu-application-platform/*"
            ],
            "Effect": "Allow",
            "Sid": "TAPEcrWorkloadScoped"
        }
    ]
}
EOF
aws iam put-role-policy --role-name tap-workload-$EKS_CLUSTER_NAME --policy-name tapWorkload --policy-document file://generated/workload-policy-$EKS_CLUSTER_NAME.json
```

Get the role ARNs and add them to the values.yml

```
aws iam get-role --role-name tap-workload-$EKS_CLUSTER_NAME --query Role.Arn --output text
aws iam get-role --role-name tap-build-service-$EKS_CLUSTER_NAME --query Role.Arn --output text
```

the build service arn should go into `tap.build.aws_tbs_role_arn` and the workload arn should go into `tap.build.aws_workload_role_arn`


### configure AWS policies for Certman and ExternalDNS

The below commands will create the policies and roles so that cert manager and external-dns can add entries to rouet53. This allows us to fully automate DNS and certs. This needs to be done for view and run clusters, so we are setting up two OIDC provdiers in the trust policy. 

export the required vars.

```
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export AWS_REGION=us-west-2
export VIEW_CLUSTER_NAME=$(cat tanzu-cli/values/values.yml| yq .clusters.view.name)
export VIEW_OIDCPROVIDER=$(aws eks describe-cluster --name $VIEW_CLUSTER_NAME --region $AWS_REGION --output json | jq '.cluster.identity.oidc.issuer' | tr -d '"' | sed 's/https:\/\///')
export RUN_CLUSTER_NAME=$(cat tanzu-cli/values/values.yml| yq .clusters.run.name)
export RUN_OIDCPROVIDER=$(aws eks describe-cluster --name $RUN_CLUSTER_NAME --region $AWS_REGION --output json | jq '.cluster.identity.oidc.issuer' | tr -d '"' | sed 's/https:\/\///')
```


create the trust policy and role. This trust policy will include both clusters. 

```

cat << EOF > generated/dns-trust.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${VIEW_OIDCPROVIDER}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${VIEW_OIDCPROVIDER}:sub": "system:serviceaccount:external-dns:external-dns",
                    "${VIEW_OIDCPROVIDER}:aud": "sts.amazonaws.com"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${RUN_OIDCPROVIDER}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${RUN_OIDCPROVIDER}:sub": "system:serviceaccount:external-dns:external-dns",
                    "${RUN_OIDCPROVIDER}:aud": "sts.amazonaws.com"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${VIEW_OIDCPROVIDER}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${VIEW_OIDCPROVIDER}:sub": "system:serviceaccount:cert-manager:cert-manager",
                    "${VIEW_OIDCPROVIDER}:aud": "sts.amazonaws.com"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${RUN_OIDCPROVIDER}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${RUN_OIDCPROVIDER}:sub": "system:serviceaccount:cert-manager:cert-manager",
                    "${RUN_OIDCPROVIDER}:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
EOF

aws iam create-role --role-name tap-dns-role --assume-role-policy-document file://generated/dns-trust.json

```

create the permissions policy and attach to the role

```
cat << EOF > generated/dns-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListHostedZonesByName",
        "route53:ListTagsForResource"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF
aws iam put-role-policy --role-name tap-dns-role --policy-name tap-dns --policy-document file://generated/dns-policy.json

```


get the role ARN and add it to the values file. This should be added as `dnsRoleArn` at the top level of the values file. 

```
aws iam get-role --role-name tap-dns-role --query Role.Arn --output text
```

Update the external helm chart values to use the ARN.

```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/overlays/dns.yml -f infra-gitops/apps/base/external-dns/install.yml --output-files infra-gitops/apps/base/external-dns
```

Update the cert manager overlay to use the new roel ARN

```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/overlays/certman-arn.yml -f infra-gitops/apps/base/tap-overlays/cert-manager-arn.yml --output-files infra-gitops/apps/base/tap-overlays
```


### Setup a hosted zone

If you do not already have a zone in route53 follow these steps.

```
aws route53 create-hosted-zone --name "tapmc.<yourdomain>.com." --caller-reference "external-dns-test-$(date +%s)"
```

Make a note of the nameservers that were assigned to your new zone.

```
ZONE_ID=$(aws route53 list-hosted-zones-by-name | jq --arg name "tapmc.aws.warroyo.com." -r '.HostedZones | .[] | select(.Name=="\($name)") | .Id')

aws route53 list-resource-record-sets --output text \
 --hosted-zone-id $ZONE_ID --query \
 "ResourceRecordSets[?Type == 'NS'].ResourceRecords[*].Value | []" | tr '\t' '\n'
```

If using your own domain that was registered with a third-party domain registrar, you should point your domain's name servers to the values in the from the list above. Please consult your registrar's documentation on how to do that.


## Setup infra gitops

This step configures the cluster group to use this git repo as the source for flux, specifically the `infra-gitops` folder. The gitops setup is done at the cluster group level so we don't need to individually bootstrap every cluster. Thsi allows us to easily install things like external dns, cluster issuers,tap overlays, workloads and deliverables. Really it can be used to add anythign to your clusters through gitops.  

before creating the TMC objects, you will need to rename the folders in `infra-gitops/clusters` to match your cluster names. Also if your cluster group name is different than `tap-mc` you will need to rename the folder in `infra-gitops/clustergroups` along with the paths in the `infra-gitops/clustergroups/<group-name>/base.yml`.

create the gitrepo in TMC

```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cd/git-repo-template.yml > generated/gitrepo.yaml
tanzu tmc continuousdelivery gitrepository create -f generated/gitrepo.yaml -s clustergroup
```

create the base kustomization that will bootstrap the clusters and setup any initial infra.


```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cd/kust-template.yml > generated/kust.yaml
tanzu tmc continuousdelivery kustomization create -f generated/kust.yaml -s clustergroup
```

at this point clusters should start syncing in multiple kustomizations. You can check their status using the below command. there will be some in a failed state until the TAP install is done. 

```
kubectl get kustomizations -A
```



## Install TAP


### Add the cluster url and ca to the values file

We need to pull back this info from the cli and update our values file. This is used so the view cluster can connect to the other clusters. 

run this for build and run profiles

**Build:**

```
export PROFILE=build
export AGENT_NAME=$(ytt --data-values-file tanzu-cli/values --data-value profile=$PROFILE  -f tanzu-cli/overlays/agentname.yml| yq .agent)
tanzu tmc cluster kubeconfig get $AGENT_NAME -m eks -p eks | ytt --data-values-file - --data-value profile=$PROFILE -f tanzu-cli/overlays/clusterdetails.yml -f tanzu-cli/values/values.yml --output-files tanzu-cli/values
```

**Run:** 

```
export PROFILE=run
export AGENT_NAME=$(ytt --data-values-file tanzu-cli/values --data-value profile=$PROFILE  -f tanzu-cli/overlays/agentname.yml| yq .agent)
tanzu tmc cluster kubeconfig get $AGENT_NAME -m eks -p eks | ytt --data-values-file - --data-value profile=$PROFILE -f tanzu-cli/overlays/clusterdetails.yml -f tanzu-cli/values/values.yml --output-files tanzu-cli/values
```

### Create the TAP solution

The below command with generate our TMC TAP install file from the values and then create the solution. This will start the install across all clusters. you can check the status in the TMC UI.

```
export TAP_NAME=$(cat tanzu-cli/values/values.yml| yq .tap.name)
ytt --data-values-file tanzu-cli/values -f tanzu-cli/tap/tap-template.yml > generated/tap.yaml
tanzu tmc tanzupackage tap create -n $TAP_NAME -f generated/tap.yaml
```


### Update the TAP solution

You can change the values in the values.yml and update the solution through the cli as well.

```
export TAP_NAME=$(cat tanzu-cli/values/values.yml| yq .tap.name)
tanzu tmc tanzupackage tap get -n $TAP_NAME -o yaml | sed '1d' | ytt --data-values-file - --data-values-file tanzu-cli/values -f tanzu-cli/overlays/generation.yml -f tanzu-cli/tap/tap-template.yml > generated/tap.yaml

```
