## Overview 

This guide decribe how to prepare a Kubernetes cluster to enable 
the deployment of Nextflow pipelines using Nextflow Tower.

## Requirements 

To allow Tower to operate with your Kubernetes cluster you will need: 

1. The cluster server URL and certificate or the vendor specific credentials 
2. A Kubernetes *namespace* and *service account* that allows the execution of Nextflow pods
3. A [ReadWriteMany](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) 
enabled shared file system used as scratch storage for the pipeline execution.

## Cluster preparation 

The following steps describe the operations required to prepare your Kubernetes cluster 
in order to enable to deployment of Nextflow pipelines using Tower. 

For the sake of this guide it assumed the Kubernetes cluster is already configured and 
you have administration permissions. 


### 0. Verify connection 

Make sure you are able to connect your Kubernetes cluster using the command in a shell terminal: 

```
kubectl cluster-info
``` 

### 1. Namespace creation

Even though this is not mandatory, it is generally recommendable to create a separate 
Kubernetes namespace for your Tower deployment. Use the command below to create a new 
Kubernetes namespace:

```
kubectl create ns <YOUR-NAMESPACE> 
```        

Replace the string `<YOUR-NAMESPACE>` with a name of your choice. 
We'll use `tower-nf` for the sake of this guide.

Finally switch to the new workspace using the command below: 

```
kubectl config set-context --current --namespace=<YOUR-NAMESPACE> 
```

e.g. 

```
kubectl config set-context --current --namespace=tower-nf
```

### 2. Service account & role creation 

This step creates a service account and the corresponding role to allow Tower to 
operate properly. 

Create the required resources and permissions file copy & pasting the following content in your 
terminal: 
 
```yaml
cat > config-rbac.yaml << EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tower-launcher-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tower-launcher-role
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/status", "pods/log", "jobs", "jobs/status", "jobs/log"]
    verbs: ["get", "list", "watch", "create", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tower-launcher-rolebind
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: tower-launcher-role
subjects:
  - kind: ServiceAccount
    name: tower-launcher-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tower-launcher-userbind
subjects:
  - kind: User
    name: tower-launcher-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: tower-launcher-role
  apiGroup: rbac.authorization.k8s.io
...
EOF
```

Then apply the policy to your cluster using this command: 

``` 
kubectl apply -f config-rbac.yaml
```

This creates a service account named `tower-launcher-sa` that will used by 
Tower to operate with the Kubernetes cluster and allow Nextflow to submit 
the pipeline jobs.

Use this service account name when setting up the Tower computer environment
for this Kubernetes cluster.

### 3. Storage configuration 

This step heavily depends on the storage options available in your infra or 
provided the by the cloud vendor. 

Tower requires the use of a *ReadWriteMany* storage mounted in all computing 
nodes where the pods will be dispatched.  

Possible supported solution includes NFS server, GlusterFS, CephFS, Amazon FSx
between the others. A comprehensive list is available at 
[this link](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes).    

The setup of such storage is beyond the scope of this guide. 
Ask your cluster administrator for more details.  

This guides shows how to configure a container based NFS server using for demoing purpose. 

__NOTE__: Do not use this for production deployment. 

Copy and paste the follwing Kubernetes resources definition in your schell terminal 
and press enter. It creates a file named `nfs-server.yaml`:

```
cat > nfs-server.yaml << EOF
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-server
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
spec:
  replicas: 1
  selector:
    matchLabels:
      role: nfs-server
  template:
    metadata:
      labels:
        role: nfs-server
    spec:
      containers:
      - name: nfs-server
        image: gcr.io/google_containers/volume-nfs:0.8
        ports:
          - name: nfs
            containerPort: 2049
          - name: mountd
            containerPort: 20048
          - name: rpcbind
            containerPort: 111
        securityContext:
          privileged: true
        volumeMounts:
          - mountPath: /exports
            name: vol-1
      volumes:
        - name: vol-1
          persistentVolumeClaim:
            claimName: nfs-server
---
apiVersion: v1
kind: Service
metadata:
  name: nfs-server
spec:
  ports:
    - name: nfs
      port: 2049
    - name: mountd
      port: 20048
    - name: rpcbind
      port: 111
  selector:
    role: nfs-server
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-storage
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs-server.tower-nf.svc.cluster.local
    path: "/"
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: tower-scratch
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 10Gi
...
EOF
```

Then apply it using the command: 

```
kubectl apply -f nfs-server.yaml
```

This action creates a persistent volume of 10 GB and a volume 
claim named `tower-scratch`. This name will be used in the configuration 
of Tower compute environment. 

### 4. Amazon EKS specific setting 

When operating with a Amazon EKS cluster you will need to assign 
the service role created in the previous step with AWS user that will 
be used by Tower to access to EKS cluster. 

Use the following command to modify the EKS auth configuration: 

```
kubectl edit configmap -n kube-system aws-auth
```

Once the editor is opened add the following entry: 

```yaml
  mapUsers: |
    - userarn: <AWS USER ARN>
      username: tower-launcher-user
      groups:
        - tower-launcher-role
```

Your user ARN can be found with the following command or using the 
 [AWS IAM console](https://console.aws.amazon.com/iam): 

```
aws sts get-caller-identity
```
 
__NOTE:__ The same user need to be used when specifying the AWS credentials in the 
configuration of the Tower compute environment for EKS. 


For more details check the [AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html).

The AWS user should have the following IAM policy: 

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TowerEks0",
      "Effect": "Allow",
      "Action": [
        "eks:ListClusters",
        "eks:DescribeCluster"
      ],
      "Resource": "*"
    }
  ]
}
```

### 5. Google GKE specific setting 

When config a Google GKE cluster you will need to grant the cluster access to the service account 
used to authenticate the Tower compute environment. This can be done updating the *role binding* 
as shown below:

```yaml
cat << EOF | kubectl apply -f -
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tower-launcher-userbind
subjects:
  - kind: User
    name: <IAM SERVICE ACCOUNT>
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: tower-launcher-role
  apiGroup: rbac.authorization.k8s.io
...
EOF
```

In the above snippet replace the placeholder `<IAM SERVICE ACCOUNT>` with the corresponding service account e.g.
`test-account@test-project-123456.google.com.iam.gserviceaccount.com`.

For more details refers the [Google documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control).
