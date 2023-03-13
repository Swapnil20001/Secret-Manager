# Secret-Manager

### Secret CSI  Driver

When using AWS, these information (environment variables) are usually stored in AWS Secret Manager or AWS Systems Manager Paramater Store(SSM). We can store environment variable in these AWS services and by using secrets csi driver, we can access in kubernetes cluster.
We usually need to access secrets from a pod to retrieve Datastores credentials, API Keys, etc.

To be able to use AWS Secrets Manager or AWS SSM Parameter Store from Kubernetes a bit of configuration is required.
 an OpenID Connect identity provider to be able to integrate IAM roles with Kubernetes ServiceAccounts
install Kubernetes Secrets Store CSI Driver
Install AWS Secrets and Configuration Provider (ASCP)

To create role:
Go to IAM in AWS. 

Roles —> web-identity —> choose identity provider of cluster —> audience (sts.amazonaws.com) —> select policy —>give name to role —> create role.

To create Policy:
Got to IAM in AWS

Policies —> create policy —> JSON —> attach policy in json —> netxt —> add name to policy —>  create Policy.

The following policy allows:
the retrieval of secrets from AWS Secrets Manager and AWS SSM Parameter Store
the use of a KMS key (required if the secrets are encrypted)

        {
          "Version": "2012-10-17",
          "Statement": [
              {
                  "Action": [
                      "secretsmanager:DescribeSecret",
                      "secretsmanager:GetSecretValue",
                      "ssm:DescribeParameters",
                      "ssm:GetParameter",
                      "ssm:GetParameters",
                      "ssm:GetParametersByPath"
                  ],
                  "Effect": "Allow",
                  "Resource": "*"
              },
              {
                  "Action": [
                      "kms:DescribeCustomKeyStores",
                      "kms:ListKeys",
                      "kms:ListAliases"
                  ],
                  "Effect": "Allow",
                  "Resource": "*"
              },
              {
                  "Action": [
                      "kms:Decrypt",
                      "kms:GetKeyRotationStatus",
                      "kms:GetKeyPolicy",
                      "kms:DescribeKey",
                  ],
                  "Effect": "Allow",
                  "Resource": "<KMS_KEY_ARN>"
              }
          ]
        }


Now create Service account to allow the pods to assume the IAM role.
### Service-account.yaml

        apiVersion: v1
        kind: ServiceAccount
        metadata:
          name: < name>
          namespace: <ns>
          annotations:
               eks.amazonaws.com/role-arn:  < role-arm> 


It is important to note that this service account is only available to the specified namespace where your pod is running.
Installation of  Secrets CSI Driver
TO install Secrets CSI Driver, we will be using helm. Apply following helm command to install CSI Driver.

    helm repo add secrets-store-csi-driver \ https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
    
    helm install -n kube-system csi-secrets-store \ --set syncSecret.enabled=true \ --set enableSecretRotation=true \ secrets-store-csi-driver/secrets-store-csi-driver



It will create 2 Pods in kube-system namespace.
Also check logs to know CSI driver successfully installed or not.

### Installation of AWS Secrets and Configuration Provider (ASCP) :-

To show secrets from Secrets Manager as files mounted in amazon eks pods, you can use the AWS Secrets and Configuration Provider (ASCP) for the Kubernetes Secrets Store CSI Driver

With the ASCP, you can store and manage your secrets in Secrets Manager and then retrieve them through your workloads running on Amazon EKS. 
If your secret contains multiple key/value pairs in JSON format, you can choose which ones to mount in Amazon EKS. The ASCP uses  JMESPath syntax to query the key/value pairs in your secret. The ASCP also works with Parameter store parameters.

You use IAM roles and policies to grant access to your secrets to specific Amazon EKS pods in a cluster.

To install ASCP in cluster, use following command:-

    kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml


It will also create 2 pods in kube-system namespace.


Now we need to create SecretProviderClass.


### SecretProviderClass:-
To determine which secrets the ASCP mounts in Amazon EKS as files on the filesystem, you create a SecretProviderClass YAML file. The SecretProviderClass YAML lists the secrets to mount and the file name to mount them . The SecretProviderClass must be in the same namespace as the Amazon EKS pod it references. 


        apiVersion: secrets-store.csi.x-k8s.io/v1
        kind: SecretProviderClass
        metadata:
         name: <name>
        spec:
         provider: aws
         secretObjects:
         - secretName: <secret-name from AWS> / Secrets-ARN
           type: Opaque
           data:
           - objectName: username-alias
              key: username
         parameters:
           objects: |
               - objectName: "<secrets-name-from AWS>"/Secrets-ARN
                 objectType: "secretsmanager"
                 jmesPath:
            - path: "username"
                    objectAlias: "username-alias"  # it can be custom name but should point to objectName in secretssObjects
         secretObjects:
           - secretName: <custom_name> # should point to name in secretKeyRef in env section.
             type: Opaque
             data:
               - objectName: username-alias
                 key: username-env    # it should point to key in env section.



In secret provider class manifest file, The first “secretObject” Block (below provider) tell kubelet, which secrets we want to access from AWS Secret Manager.

In parameter section, tell kubelet to which parameter ( i.e Key-value pair)  want to access it from secrets manager.

And last SecretObject block ( below the parameter), used to create secrets for cluster/ local. This secret create when we apply pod manifest file.

The secret name from this block should point to the name (-name: <>) in secretKeyRef from the environment section in pod manifest file. And key from secretobject should point to key in secretKeyRef .

In Pod:-

        env:
           - name: DB_HOST
             valueFrom:
                secretKeyRef:
                    name: < custom_name>
                    Key: username-env


We also need to mount secrets as a volume in pods.

        volumeMounts:
         - name: secrets-store
             mountPath: "/mnt/secrets-store"
             readOnly: true
         volumes:
         - name: secrets-store
           csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
              secretProviderClass: <secret-provider-class-name>

