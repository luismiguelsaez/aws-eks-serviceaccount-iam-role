
## Export variables
```
export AWS_ACCOUNT_ID=****
```
## Create EKS OIDC provider ( https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html )
### Get OIDC provider URL
```
aws eks describe-cluster --name test --query "cluster.identity.oidc.issuer" --output text
```
```
https://oidc.eks.eu-west-1.amazonaws.com/id/B3BD129E029B5A85B357FE797F46BE4D
```
### List account providers
```
aws iam list-open-id-connect-providers | grep B3BD129E029B5A85B357FE797F46BE4D
```
```
"Arn": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/B3BD129E029B5A85B357FE797F46BE4D"
```
### Create EKS cluster IAM OIDC identity ( if no one was returned from previous commands )
```
eksctl utils associate-iam-oidc-provider --cluster test --approve
```

## Create IAM policy and role ( https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html )
### Create testing bucket
```
aws s3api create-bucket --bucket test-oidc-20210202 --region eu-west-1 --create-bucket-configuration LocationConstraint=eu-west-1
```
```
{
    "Location": "http://test-oidc-20210202.s3.amazonaws.com/"
}
```
### Create IAM policy
```
aws iam create-policy --policy-name test-oidc --policy-document file://aws/iam/policy.json
```
```
{
    "Policy": {
        "PolicyName": "test-oidc",
        "PolicyId": "ANPAUWSOS3H4P3LFSQK5S",
        "Arn": "arn:aws:iam::****:policy/test-oidc",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2021-02-02T17:25:51+00:00",
        "UpdateDate": "2021-02-02T17:25:51+00:00"
    }
}
```
### Create service account role
- Through eksctl
```
eksctl create iamserviceaccount \
    --name test-oidc \
    --namespace default \
    --cluster test \
    --attach-policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/test-oidc \
    --approve \
    --override-existing-serviceaccounts
```
- Through cli
```
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
```
```
OIDC_PROVIDER=$(aws eks describe-cluster --name test --query "cluster.identity.oidc.issuer" --output text | sed -e "s/^https:\/\///")
```
```
read -r -d '' TRUST_RELATIONSHIP <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:default:test-oidc"
        }
      }
    }
  ]
}
EOF
echo "${TRUST_RELATIONSHIP}" > aws/iam/trust.json
```
```
aws iam create-role --role-name test-oidc --assume-role-policy-document file://aws/iam/trust.json --description "Test OIDC"
```
```
aws iam attach-role-policy --role-name test-oidc --policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/test-oidc
```

## Associate IAM role to service account ( https://docs.aws.amazon.com/eks/latest/userguide/specify-service-account-role.html )
### Update kubeconfig
```
aws eks update-kubeconfig --name test
```
### Create EKS cluster service account
```
cat << EOF > k8s/service-account.yml
---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: test
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::${AWS_ACCOUNT_ID}:role/test-oidc
EOF
```
```
kubectl apply -f k8s/service-account.yml
```
### Create pod with service account attached
```
kubectl run oidc-test -n default -it --rm=true --image=amazon/aws-cli --serviceaccount=test s3 ls s3://test-oidc-20210202
```
