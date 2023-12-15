# tmc-tap-quickstart

This repo is an example of how to quickly get up and running in EKS with TAP using the Tanzu CLI for TMC. The goal of this repo is to simplify all of the pieces involved in standing up TAP down to some CLI commands. The goal of this is not to entirely script the process, but rather outline and document all of the individual commands needed to set up TAP in an opinionated way on EKS using TMC. This repo does not create an iterate cluster at this time. the current focus of this repo is on the outer loop.

## Create the cluster group

```
ytt -f tanzu-cli/values.yml -f tanzu-cli/cluster-group/cg-template.yml > cg.yaml
tanzu tmc clustergroup create -f cg.yaml
rm cg.yaml
```
## enable helm and flux

```
tanzu tmc continuousdelivery enable -g tap-mc -s clustergroup
tanzu tmc helm enable -g tap-mc -s clustergroup
```
## Create cluster group  secrets

first generate a PAT in github using the legacy token method. give the pat access to all repo privileges

```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/github-pat-template.yml > pat-secret.yaml
tanzu tmc secret create -f pat-secret.yaml -s clustergroup
rm pat-secret.yaml

ytt --data-values-file tanzu-cli/values -f tanzu-cli/secrets/github-pat-export-template.yml > pat-export.yaml
tanzu tmc secret export create -f pat-export.yaml -s clustergroup
```

## Create the clusters

```
ytt -f tanzu-cli/clusters/<profile-name>.yml -f tanzu-cli/clusters/cluster-template.yml > <profile-name>-cluster.yaml

ytt -f tanzu-cli/clusters/<profile-name>.yml -f tanzu-cli/clusters/nodepool-template.yml > <profile-name>-np.yaml

tanzu tmc ekscluster create -f <profile-name>-cluster.yaml
tanzu tmc ekscluster nodepool create -f <profile-name>-np.yaml                                                                 
```

## asscoiate the OIDC url with a provider

run this for all three clusters

```
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve
```

## Create ECR repos 

[official docs](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/install-aws-resources.html#create-the-workload-container-repositories-5)


```
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export AWS_REGION=us-west-2

aws ecr create-repository --repository-name tanzu-application-platform/tanzu-java-web-app-dev --region $AWS_REGION
aws ecr create-repository --repository-name tap-build-service --region $AWS_REGION

```


## Create the AWS roles/policies for ECR

[official docs](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.7/tap/install-aws-resources.html#create-iam-roles-6)


export the required vars
```
export EKS_CLUSTER_NAME=$(cat tanzu-cli/values.yml| yq .clusters.build.name)
export OIDCPROVIDER=$(aws eks describe-cluster --name $EKS_CLUSTER_NAME --region $AWS_REGION --output json | jq '.cluster.identity.oidc.issuer' | tr -d '"' | sed 's/https:\/\///')
```

create the workload trust policy specific to the cluster's OIDC endpoint 

```
cat << EOF > workload-trust-policy-$EKS_CLUSTER_NAME.json
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

aws iam create-role --role-name tap-workload-$EKS_CLUSTER_NAME --assume-role-policy-document file://workload-trust-policy-$EKS_CLUSTER_NAME.json
rm workload-trust-policy-$EKS_CLUSTER_NAME.json
```

create the workload policy specific to the account.

```
cat << EOF > workload-policy.json
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
aws iam put-role-policy --role-name tap-workload-$EKS_CLUSTER_NAME --policy-name tapWorkload --policy-document file://workload-policy.json
rm 
workload-policy.json
```

get the role ARN to be used in the values.

```
aws iam get-role --role-name tap-workload-$EKS_CLUSTER_NAME --query Role.Arn --output text
```

## configure AWS policies for Certman and ExternalDNS

this needs to be done for view and run clusters, so we are setting up two OIDC provdiers in the trust

export the required vars
```
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export AWS_REGION=us-west-2
export VIEW_CLUSTER_NAME=$(cat tanzu-cli/values/values.yml| yq .clusters.view.name)
export VIEW_OIDCPROVIDER=$(aws eks describe-cluster --name $VIEW_CLUSTER_NAME --region $AWS_REGION --output json | jq '.cluster.identity.oidc.issuer' | tr -d '"' | sed 's/https:\/\///')
export RUN_CLUSTER_NAME=$(cat tanzu-cli/values/values.yml| yq .clusters.run.name)
export RUN_OIDCPROVIDER=$(aws eks describe-cluster --name $RUN_CLUSTER_NAME --region $AWS_REGION --output json | jq '.cluster.identity.oidc.issuer' | tr -d '"' | sed 's/https:\/\///')
```
create the trust

```

cat << EOF > dns-trust.json
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

aws iam create-role --role-name tap-dns-role --assume-role-policy-document file://dns-trust.json
rm dns-trust.json

```

create the policy and attach to the role

```
cat << EOF > dns-policy.json
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
aws iam put-role-policy --role-name tap-dns-role --policy-name tap-dns --policy-document file://dns-policy.json

```


get the role arn to be used in the values

```
aws iam get-role --role-name tap-dns-role --query Role.Arn --output text
```

update the role arn in the external dns helm chart install. This is in `infra-gitops/apps/base/external-dns/install.yml`
Update the cert manager role arn in the values file `infra-gitops/apps/base/tap-overlays/cert-manager-arn.yml`

## Setup a hosted zone

if you do not already have a zone in route53 follow these steps.

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

create the git repo on the cluster group

```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cd/git-repo-template.yml > gitrepo.yaml
tanzu tmc continuousdelivery gitrepository create -f gitrepo.yaml -s clustergroup
rm gitrepo.yaml
```

create the base kustomization that will bootstrap the clusters and setup any initial infra.


```
ytt --data-values-file tanzu-cli/values -f tanzu-cli/cd/kust-template.yml > kust.yaml
tanzu tmc continuousdelivery kustomization create -f kust.yaml -s clustergroup
rm kust.yaml
```


## Install TAP

```
export TAP_NAME=$(cat tanzu-cli/values/values.yml| yq .tap.name)
ytt --data-values-file tanzu-cli/values -f tanzu-cli/tap/tap-template.yml > tap.yaml



tanzu tmc tanzupackage tap create -n $TAP_NAME -f tap.yaml
rm tap.yaml
```