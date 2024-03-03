# Kubernetes Cluster Autoscaler for AWS EKS

In this article, ideally we will deploy a web app onto a cloud production Kubernetes cluster that can be consumed by users on the public internet. Because this is deployed in a Kubernetes environment the orchestration of the app containers is done using the Kubernetes tech stack. Though there are many flavors of managed Kubernetes services, for this specific project we will leverage AWS's managed Kubernetes service AWS Elastic Kubernetes Service (EKS) for their cloud production Kubernetes cluster.

Instead of growing the discussion from scratch from picking up an app, build its image into Docker Images (such as in both [x86_64 and arm64](https://medium.com/p/1b54bd938d96/edit) formats with Docker Buildx tool, push the images to DockerHub registry), we focus more on EKS and assuming some sort of apps were installed in the Kubernetes cluster instances. We will apply a dummy stress test by increasing the unit of [CPU request](https://github.com/FariusGitHub/kubernetes/blob/main/images/05-image11.png) which forces Kubernetes cluster to autoscale to bring more nodes. In short we will cover the right side of below chart.
![](/images/14-image01.png)


Using my teaching assistant lecture notes [Dawei](https://www.linkedin.com/in/dawei-zhang-96b669160/), I [documented](https://github.com/FariusGitHub/eks) his approach and I would like to summarize the autoscaling process as follow.

In the beginning we need to create an AWS EKS cluster from this command for example.

```sh
eksctl create cluster \
--name EKS-Lab \
--version 1.28 \
--region us-east-1 \
--nodegroup-name linux-nodes \
--node-ami-family AmazonLinux2 \
--node-type m5.xlarge \
--nodes 3 \
--nodes-min 1 \
--nodes-max 10 \
--managed \
--with-oidc \
--ssh-access \
--asg-access \
--ssh-public-key ~/.ssh/id_rsa.pub
```

We may need to install [eksctl](https://eksctl.io/installation/) in the first place to run above command.

## What we will simulate the load in the nutshell is as follow <br> 1. Initially create a kubernetes cluster says with a nodes with 3 pods <br> 2. Imitate low load by introducing unneeded pods to turn off after 3 minutes <br> 3. Immitate high load that requires 20 pods, each need 1 CPU & 128MB memory <br>

## FIRST CASE: INITIALLY CREATE A NODE WITH 3 PODS

By the end of this exercise we will see 3 nodes was created following above command.
![](/images/14-image11.png)
We could also run the command through a yaml file like below
```sh
$ cat cluster.yaml
  apiVersion: eksctl.io/v1alpha5
  kind: ClusterConfig
  metadata:
    name: EKS-Lab
    region: us-east-1
    version: "1.28"

  iam:
    withOIDC: true

  managedNodeGroups:
    - name: linux-nodes
      instanceType: m5.xlarge
      minSize: 1
      maxSize: 10
      desiredCapacity: 3
      amiFamily: AmazonLinux2
      ssh:
        publicKeyPath: ~/.ssh/id_rsa.pub
      iam:
        withAddonPolicies:
          autoScaler: true

$ eksctl create cluster -f cluster.yaml
```
EKS cluster normally takes 10-15 minutes to create. If we did some mistake, it would need 10-15 minutes to delete it and need to spend another 10-15 minutes to re-create it. <br>
Below is the cloudformation which would be create by default.<br>
![](/images/14-image02.png)

The Name of EKS cluster created would be the same as from eksctl command above. Unlike cloudformation stack name which were added by some sorts of prefix and suffix.<br>
![](/images/14-image03.png)

Along with EKS cluster creation, we can see below that a role was also setup.<br>
![](/images/14-image06.png)

Most importantly we can see the node group was created.<br>
![](/images/14-image07.png)

Inside this node group we can see how the requested 3 nodes were created.<br>
![](/images/14-image04.png)

As part of EKS package, the important element is an Auto Scaling Group (ASG) which 
responsible to maintain necessary EC2.<br>
![](/images/14-image08.png)
And we can see inside this group that those 3 nodes represented by 3 EC2 were created.<br>
![](/images/14-image05.png)

To access these instances easily from our local machine, we may need to update current kubernetes context, than allow use to monitor the cluster with K3S for example locally.<br>
![](/images/14-image09.png)

## SECOND CASE: TURNING OFF UNNEEDED PODS AFTER 3 MINUTES
By the end of this exercise we will have 2 unnecessary pods deleted like below.
![](/images/14-image12.png)
To do so we can download and run a yaml file below for example.<br>
```txt
cat cluster-autoscaler-autodiscovery.yaml
  ---
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    labels:
      k8s-addon: cluster-autoscaler.addons.k8s.io
      k8s-app: cluster-autoscaler
    name: cluster-autoscaler
    namespace: kube-system
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: cluster-autoscaler
    labels:
      k8s-addon: cluster-autoscaler.addons.k8s.io
      k8s-app: cluster-autoscaler
  rules:
    - apiGroups: [""]
      resources: ["events", "endpoints"]
      verbs: ["create", "patch"]
    - apiGroups: [""]
      resources: ["pods/eviction"]
      verbs: ["create"]
    - apiGroups: [""]
      resources: ["pods/status"]
      verbs: ["update"]
    - apiGroups: [""]
      resources: ["endpoints"]
      resourceNames: ["cluster-autoscaler"]
      verbs: ["get", "update"]
    - apiGroups: [""]
      resources: ["nodes"]
      verbs: ["watch", "list", "get", "update"]
    - apiGroups: [""]
      resources:
        - "namespaces"
        - "pods"
        - "services"
        - "replicationcontrollers"
        - "persistentvolumeclaims"
        - "persistentvolumes"
      verbs: ["watch", "list", "get"]
    - apiGroups: ["extensions"]
      resources: ["replicasets", "daemonsets"]
      verbs: ["watch", "list", "get"]
    - apiGroups: ["policy"]
      resources: ["poddisruptionbudgets"]
      verbs: ["watch", "list"]
    - apiGroups: ["apps"]
      resources: ["statefulsets", "replicasets", "daemonsets"]
      verbs: ["watch", "list", "get"]
    - apiGroups: ["storage.k8s.io"]
      resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities"]
      verbs: ["watch", "list", "get"]
    - apiGroups: ["batch", "extensions"]
      resources: ["jobs"]
      verbs: ["get", "list", "watch", "patch"]
    - apiGroups: ["coordination.k8s.io"]
      resources: ["leases"]
      verbs: ["create"]
    - apiGroups: ["coordination.k8s.io"]
      resourceNames: ["cluster-autoscaler"]
      resources: ["leases"]
      verbs: ["get", "update"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: cluster-autoscaler
    namespace: kube-system
    labels:
      k8s-addon: cluster-autoscaler.addons.k8s.io
      k8s-app: cluster-autoscaler
  rules:
    - apiGroups: [""]
      resources: ["configmaps"]
      verbs: ["create", "list", "watch"]
    - apiGroups: [""]
      resources: ["configmaps"]
      resourceNames: ["cluster-autoscaler-status", "cluster-autoscaler-priority-expander"]
      verbs: ["delete", "get", "update", "watch"]

  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: cluster-autoscaler
    labels:
      k8s-addon: cluster-autoscaler.addons.k8s.io
      k8s-app: cluster-autoscaler
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-autoscaler
  subjects:
    - kind: ServiceAccount
      name: cluster-autoscaler
      namespace: kube-system

  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: cluster-autoscaler
    namespace: kube-system
    labels:
      k8s-addon: cluster-autoscaler.addons.k8s.io
      k8s-app: cluster-autoscaler
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: cluster-autoscaler
  subjects:
    - kind: ServiceAccount
      name: cluster-autoscaler
      namespace: kube-system

  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: cluster-autoscaler
    namespace: kube-system
    labels:
      app: cluster-autoscaler
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: cluster-autoscaler
    template:
      metadata:
        labels:
          app: cluster-autoscaler
        annotations:
          prometheus.io/scrape: 'true'
          prometheus.io/port: '8085'
      spec:
        priorityClassName: system-cluster-critical
        securityContext:
          runAsNonRoot: true
          runAsUser: 65534
          fsGroup: 65534
          seccompProfile:
            type: RuntimeDefault
        serviceAccountName: cluster-autoscaler
        containers:
          - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.2
            name: cluster-autoscaler
            resources:
              limits:
                cpu: 100m
                memory: 600Mi
              requests:
                cpu: 100m
                memory: 600Mi
            command:
              - ./cluster-autoscaler
              - --v=4
              - --stderrthreshold=info
              - --cloud-provider=aws
              - --skip-nodes-with-local-storage=false
              - --expander=least-waste
              - --scale-down-unneeded-time=3m
              - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/EKS-Lab
            volumeMounts:
              - name: ssl-certs
                mountPath: /etc/ssl/certs/ca-certificates.crt # /etc/ssl/certs/ca-bundle.crt for Amazon Linux Worker Nodes
                readOnly: true
            imagePullPolicy: "Always"
            securityContext:
              allowPrivilegeEscalation: false
              capabilities:
                drop:
                  - ALL
              readOnlyRootFilesystem: true
        volumes:
          - name: ssl-certs
            hostPath:
              path: "/etc/ssl/certs/ca-bundle.crt"

```
This yaml file was inspired from this [template](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml) where some changes are highlighted as below.
![](/images/14-image00.png)

## THIRD CASE: ADDING LOAD WITH 20 PODS WITH 1 CPU and 128MB MEMORY EACH
On top a running instance above, additonal 20 pod replicas will be added. By the end we will see something like below.
![](/images/14-image13.png)

herewith how nginx.yaml looks like and how we execute the file

```txt
$ cat nginx.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx
  spec:
    selector:
      matchLabels:
        app: nginx
    replicas: 20
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx
          ports:
          - containerPort: 80
          resources:
            requests:
              cpu: 1000m
              memory: 128Mi
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx
    labels:
      app: nginx
  spec:
    ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
    selector:
      app: nginx
    type: LoadBalancer

$ kubectl create -f nginx.yaml
```

Take a look at instance id i-06e42f4b47add3fa. On top of this existing instance EKS created another new 6 instances. Herewith is a simple explaination.

We first need to calculate the total CPU and memory requirements for all pods. <br>

Total CPU required = 20 pods * 1 CPU = 20 CPUs<br>
Total memory required = 20 pods * 128MB = 2560MB<br>

Now, let's check the specifications of an AWS m5.xlarge instance:<br>
- 4 vCPUs<br>
- 16GB memory<br>

Since each m5.xlarge instance has 4 vCPUs and 16GB memory, we can calculate how many instances are needed to meet the requirements:<br>

Number of instances needed for CPU = Total CPU required / CPUs per instance = 20 CPUs / 4 CPUs per instance = 5 instances <br>
Number of instances needed for memory = Total memory required / memory per instance = 2560MB / 16000MB per instance = 0.16 instances<br>

Since we cannot have a fraction of an instance, we need to round up to the nearest whole number. Therefore, we would need a total of 6 AWS m5.xlarge instances to meet the requirements for 20 pods.<br>

# SUMMARY
AWS EKS is a good tool to orchestrate kubernetes cluster on AWS platform. In this blog I bring three situations since the cluster was created, some measure taken to reduce cost and finally to simulate a spike of demand (let's say web traffic during holiday season) all with tweaking the hardware alone without installing the real app for the stress test.
![](/images/14-image10.png)