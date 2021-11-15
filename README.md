# Integration of AWS EKS with Secret Manager POC
Here we are going to Setup an AWS EKS using eksctl and get secrets injected into container directly from Secrets Manager.

## AWS CLI

#### Install latest awscli version

```
pip3 install --upgrade --user awscli 
```


#### eksctl

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
```
sudo mv /tmp/eksctl /usr/local/bin
```
```
eksctl version
```

#### Kubernetes Cluster (AWS EKS)

Let’s create the Kubernetes Cluster using eksctl (**Update the cluster name accordingly**)

```
eksctl create cluster --name <Cluster Name> --version 1.17 --region us-west-2 --fargate
```
### Associate IAM oidc provider which will be required by the iam trust policy in later stage

```
eksctl utils associate-iam-oidc-provider --cluster <Cluster Name> --approve
```

#### Deploying AWS Secrets Admission Controller

AWS Secrets Admission Controller can be deployed via a Helm chart. The Helm chart creates the following Kubernetes objects:

* A Kubernetes deployment running the admission controller.
* A Kubernetes service that exposes the above deployment.
* A Kubernetes secret that contains the TLS certificates for the admission controller.
* A MutatingWebhookConfiguration [object](https://v1-18.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#mutatingwebhookconfiguration-v1-admissionregistration-k8s-io).

#### Steps to Setup AWS Secrets Admission Controller

* Add the Helm repository and update the repository

```
helm repo add secret-inject https://aws-samples.github.io/aws-secret-sidecar-injector/
```
```
helm repo update
```
* Install the AWS Secret Controller by installing the Helm chart.

```
helm install secret-inject secret-inject/secret-inject
```
* Verify that objects were created successfully

```
$ kubectl get mutatingwebhookconfiguration

###Expected Output
NAME CREATED AT
aws-secret-inject 2015-01-17T14:20:40Z
```

#### Create the Secrets in Secret Manager

For This POC we will be deploying an application **webserver** which needs a database password to run.

We have already created a secret in AWS Secrets Manager. If you need directions on how to create secrets in AWS Secrets Manager, please refer to the AWS [documentation](https://docs.aws.amazon.com/es_es/secretsmanager/latest/userguide/tutorials_basic.html).

```
$ aws secretsmanager list-secrets 

###Expected Output
SecretList:- ARN: arn:aws:secretsmanager:us-east-1:123456789012:secret:database-password-mlPtsR
  Description: Password for the MySQL database
  LastChangedDate: '2019-03-12T03:11:29.523000+00:00'
  Name: database-password
  SecretVersionsToStages:
    35z9768l-644x-4592-a856-6c747be567d9:
    - AWSCURRENT
  Tags: []
  ```
  *The ARN in the output will be used later on.*
  
  **** Create an IRSA(Iam role for service account) to access secrets in AWS Secrets Manager
  
  ```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "webserversecret",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetResourcePolicy",
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret",
                "secretsmanager:ListSecretVersionIds"
            ],
            "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:database-password-hlRvvF"
        },
        {
            "Sid": "secretslists",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetRandomPassword",
                "secretsmanager:ListSecrets"
            ],
            "Resource": "*"
        }
    ]
}
  ```
Save the above Policy Json in policy.json then create the role using below command

```
aws iam create-policy --policy-name webserver-secrets-policy --policy-document file://myPolicy.json
```

Policy for Iam role is done now we have to create the iam role which our pod will assume and access the secrets.

* Step 1 is to create ENV variables for trust policy.

```
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)

OIDC_PROVIDER=$(aws eks describe-cluster --name <cluster-name> --query "cluster.identity.oidc.issuer" --output text | sed -e "s/^https:\/\///")
```
* Step 2 create policy.json(we are doing the PoC for default namespace. You can update the namespace in ```${OIDC_PROVIDER}:sub":``` for other namespaces as per requirement. )

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
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:default:webserver-service-account",
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF
echo "${TRUST_RELATIONSHIP}" > trust.json
```

* Step 3 Create an IAM role and attach the above created policy to role.

```
aws iam create-role --role-name webserver-secrets-role --assume-role-policy-document file://trust.json --description "IAM Role to access webserver secret"

aws iam attach-role-policy --role-name webserver-secrets-role --policy-arn=arn:aws:iam::123456789012:policy/webserver-secrets-policy
```

**** Creating a Kubernetes Service Account

* Create the Kubernetes Service Account.

```
kubectl create sa webserver-service-account
```
* Add an annotation for the service account that references the IAM role we created earlier. In this case, we’re using the ARN of the role we created earlier.

```
kubectl edit sa webserver-service-account

###Add annotations and save it
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/webserver-secrets-role
  creationTimestamp: "2021-10-06T00:17:04Z"
  name: webserver-service-account
  namespace: default
  resourceVersion: "33335479"
  selfLink: /api/v1/namespaces/default/serviceaccounts/webserver-service-account
  uid: eef8b19d-7bd0-4390-94ab-186a5d677fd0
secrets:
- name: webserver-service-account-token-x5t4q
```
**** Testing it

* Create a Kubernetes deployment object for the webserver.

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    run: webserver
  name: webserver
spec:
  replicas: 1
  selector:
    matchLabels:
      run: webserver
  template:
    metadata:
      annotations:
        secrets.k8s.aws/sidecarInjectorWebhook: enabled
        secrets.k8s.aws/secret-arn: arn:aws:secretsmanager:us-east-1:123456789012:secret:database-password-mlPtsR
      labels:
        run: webserver
    spec:
      serviceAccountName: webserver-service-account
      containers:
      - image: busybox:1.28
        name: webserver
        command: ['sh', '-c', 'echo $(cat /tmp/secret) && sleep 3600']
EOF
```
* For output you can check the logs of pod

```
ubectl logs webserver-66f6bb988f-x774k

{"username":"administrator","password":"myPass@678","engine":"mysql","host":"myDB1.cluster.us-east-1.rds.amazonaws.com","port":3306,"dbClusterIdentifier":"database-1"
```
**** Conclusion

This PoC enables you to access secret from AWS Secrets Manager by running an [Init](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) container during the pod start up.
