# Integrating AWS SCP with Kubernetes

this document is following [AWS Documentation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html)
[github.com/aws/secrets-store-csi-driver-provider-aws](https://github.com/aws/secrets-store-csi-driver-provider-aws)

## Prerequisites

- An AWS account

- An AWS Identity and Access Management account with permissions to modify an Secrets Manager.

- IAM roles for service accounts (IRSA) configured.
you better read [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
[serviceaccount policy](https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html)

You use the IAM roles for IRSA to limit secret access to your pods. When you set this up, the provider retrieves the pod identity and exchange this identity for an IAM role. ASCP assumes the IAM role of the pod and only retrieves secrets from Secrets Manager authorized for the pod. This prevents the container from accessing secrets intended for another container.

- Specifically you need the following IAM policy permissions:

```
secretsmanager:GetSecretValue

secretsmanager:SecretsManager:DescribeSecret

ssm:GetParameters
```

- Amazon EKS Version 1.17 or later.

- Permissions to modify your Kubernetes cluster.

- AWS CLI and kubectl installed.

- helm

- eksctl

## Installing the CSI driver

1. Install the Kubernetes secrets store container storage interface (CSI) driver using the following helm commands.

From the CLI where you have installed kubectl, run the following commands to install the CSI driver:

```
helm repo add secrets-store-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/master/charts
helm -n kube-system install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --set grpcSupportedProviders="aws"
```


Install the AWS Secrets and Configuration Provider using a deployment YAML.


```
curl -s https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml | kubectl apply -f -
```

## SecretProviderClass
Creating and deploying the SecretProviderClass custom resource.

To use the Secrets Store CSI Driver, you must create a SecretProviderClass custom resource. The custom resource provides driver configurations and provider-specific parameters to the CSI driver. The SecretProviderClass resource should contain the following components, at a minimum:

```

    apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
    kind: SecretProviderClass
    metadata:
      name: aws-secrets
    spec:
      provider: aws
      parameters:
        objects: | 
            - objectName: "MySecret"
              objectType: "secretsmanager"s
            - |
            objectName: "/default/production_qa/otel_exporter_otlp_endpoint"
            objectType: "ssm"# object types: secretsmanager for secrets and ssm for configuration values
        
```

To implement ASCP, you create the ```SecretProviderClass``` with more details about retrieving configurations from Parameter Store and secrets from Secrets Manager.
The ```SecretProviderClass``` must be in the same namespace as the referenced pod. 


### The parameters section contains the details of the mount request and has one of three fields:

- objects - A string containing a YAML declaration of the secrets to mount.
```
parameters:
  objects: |
      - objectName: "MySecret"
        objectType: "secretsmanager"
        
```
- region (Optional) - Specifies the AWS region to use when retrieving secrets from Secrets Manager or Parameter Store. If you don't use this field, the provider lookups the region from the annotation on the node. This lookup adds overhead to mount requests so clusters using large numbers of pods benefit from providing the region here.

- pathTranslation (Optional) - Specifies a substitution character to use if you add the path separator character, such as slash on Linux, in the file name. If a Secret or parameter name contains the path separator, failures occur when the provider tries to create a mounted file using the name. When not specified, the underscore character is used, and My/Path/Secret mounts as My_Path_Secret. This pathTranslation value can either be the string False or a single character string. When set to False, no character substitution is performed.

### The objects field of SecretProviderClass can contain the following sub-fields:

- objectName(Required) - Specifies the name of the secret or parameter.
   For Secrets Manager, the SecretId parameter can be either the friendly name or full ARN of the secret.
   For SSM Parameter Store, this must be the Name of the parameter and cannot contain a full ARN.

- objectType - Optional if you use a Secrets Manager ARN for the objectName.
 If you don't use a Secrets Manager ARN, SecretProviderClass requires the field and can be either secretsmanager or ssmparameter.

objectAlias(Optional) - Specifies the file name to mount the secret. If not specified, it defaults to objectName.

objectVersion(Optional) - Recommended to not use this field as it requires updating if you update the secret.

objectVersionLabel(Optional) - Specifies the alias used for the version. SecretProviderClass uses the most recent version by default.



## Create the AWS Policy to be Binded to the ServiceAccount
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": [
                "arn:aws:secretsmanager:us-east-1:<account-id>:secret:MySecret"
              ]
        },
      {
        "Effect": "Allow",
        "Action": [
          "ssm:GetParameters"
        ],
        "Resource": [
          "arn:aws:ssm:us-east-1:<account-id>:parameter/default/production_qa/otel_exporter_otlp_endpoint"
        ]
      }
    ]
  }
```


## Create an IAM role for a service account
Create an IAM role for your service account. You can use eksctl, the AWS Management Console, or the AWS CLI to create the role.

**Prerequisites**

- If using the AWS Management Console or AWS CLI to create the role, then you must have an existing IAM OIDC provider for your cluster. For more information, see Create an IAM OIDC provider for your cluster.

- An existing IAM policy that includes the permissions for the AWS resources that your service account needs access to. For more information, see Create an IAM policy.

- You can create the IAM role with eksctl, the AWS Management Console, or the AWS CLI. Select the tab with the name of the tool that you want to create the role with.

Before usage we must ensure the existing of OIDC endpoint 
Instructions are [here](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)




  
```
eksctl create iamserviceaccount \
    --name <service_account_name> \
    --namespace <service_account_namespace> \
    --cluster <cluster_name> \
    --attach-policy-arn <IAM_policy_ARN> \
    --approve \
    --override-existing-serviceaccounts
```

I will use it:

```
eksctl create iamserviceaccount \
    --name nginx-deployment-sa \
    --namespace default \
    --cluster tekton-ci \
    --attach-policy-arn arn:aws:iam::<account_id>:policy/k8s-secrets-policy-test \
    --approve \
    --override-existing-serviceaccounts
```


Now we have the ServiceAccount and the Binding with the Policy
```
➜  ~  kubectl get secret | grep  nginx-deployment-sa
nginx-deployment-sa-token-cn8v2          kubernetes.io/service-account-token   3      5m26s
```


we can look inside via:

```
kubectl edit secret nginx-deployment-sa-token-cn8v2

apiVersion: v1
data:
  ca.crt: xxxxx==
  token: xxxxx==
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: nginx-deployment-sa
    kubernetes.io/service-account.uid: xxxxx-xxxxx-yyyyyy-zzzzzzzz
  creationTimestamp: "2021-06-02T12:25:06Z"
  name: nginx-deployment-sa-token-cn8v2
  namespace: default
  resourceVersion: "6846509"
  selfLink: /api/v1/namespaces/default/secrets/nginx-deployment-sa-token-cn8v2
  uid: abcdefg-hijklmn--zzzzzzzz
type: kubernetes.io/service-account-token
```



## SecretProviderClass

Below we can see the way to create kubernetes Secret

the part including the ```secretObjects``` would define the specific parameters from SSM or Secret Manager
```
secretObjects:
    - secretName: my-aws-secret  # the k8s secret name
      type: Opaque
      data:
        - key: my-secret
          objectName: MySecret  # reference the corresponding parameter
          
        - key: OTEL_EXPORTER
          objectName: "myssm"  # reference the corresponding parameter
```

In the ```Parameters``` section we use specifically for the SSM the 
```objectType: "ssmparameter"``` and the ```objectAlias``` in the form:

```
objectAlias: "myssm"
```

Note the SSM pramater  ```objectName``` is the SSM name and nor ARN

```
- objectName: "/default/production_qa/otel_exporter_otlp_endpoint"
```


In case of a secret it is easier:
The type is ```objectType: "secretsmanager"``` and the ```objectName``` can be either the SecretNAme or it's ARN

```
- objectName: "MySecret"
  objectType: "secretsmanager"
```

```
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  secretObjects:
    - secretName: my-aws-secret  # the k8s secret name
      type: Opaque
      data:
        - key: my-secret
          objectName: MySecret  # reference the corresponding parameter
        - key: my-secret2
          objectName: my-secret2  # reference the corresponding parameter
          
        - key: OTEL_EXPORTER
          objectName: "myssm"  # reference the corresponding parameter
  
        - key: id_rsa
          objectName: "tekton_ssh_id_rsa"
          
  parameters:
    objects: | 
        - objectName: "MySecret"
          objectType: "secretsmanager" 
        - objectName: "/default/production_qa/otel_exporter_otlp_endpoint"
          objectType: "ssmparameter"
          objectAlias: "myssm"
        - objectName: "tekton_ssh_id_rsa"
          objectType: "secretsmanager"
        - objectName: "my-secret2"
          objectType: "secretsmanager"
        

```


## Creation of Secret automatically from a secret
Some types we need to create a specific Secret with annotaions or other additios that cannot be created with the AWS ACSP CSI Driver.

The solution for that is to create the secret you need from the given secrets.
We can do that via a POD or with Tekton Task.
Tekton Catalog has a specific Task for ```kubectl``` usage the Task is ```kubernetes-actions``` you can found it [here](https://hub.tekton.dev/tekton/task/kubernetes-actions).
In this example we will use it with the original secret created by the SecretProviderClass ```my-aws-secret```
and create a new one ```private-git-repos```.


The basic Credentials neede for that are for the creation of secrets.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-secrets-sa
secrets:
  - name: my-aws-secret

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: tekton-secrets-sa-role
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "delete", "patch", "create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tekton-secrets-sa-rb
subjects:
  - kind: ServiceAccount
    name: tekton-secrets-sa
roleRef:
  kind: Role
  name: tekton-secrets-sa-role
  apiGroup: rbac.authorization.k8s.io
---
```

### We should install the Task from the Tekton Hub

```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/kubernetes-actions/0.2/kubernetes-actions.yaml

```

Now we can run the Task via TaskRun
This would use the mounted secrets for ```tekton_github_username``` and ```tekton_github_password``` to get the username ands password.
 
```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: configure-secrets-task
spec:
  serviceAccountName: tekton-secrets-sa
  taskRef:
    name: kubernetes-actions
  params:
    - name: script
      value: |
        kubectl get pods
        cat <<EOF | kubectl apply -f -
        apiVersion: v1
        kind: Secret
        metadata:
          name: private-git-repos
          annotations:
            tekton.dev/git-0: https://github.com # Described below
        type: kubernetes.io/basic-auth
        stringData:
          username:  $(cat username)
          password:  $(cat password)
        EOF
  workspaces:
    - name: manifest-dir
      secret:
        secretName: my-aws-secret
```
# aws-ascp-secrets-csi-driver-example
# aws-ascp-secrets-csi-driver-example
