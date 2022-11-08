## PERSISTING DATA IN KUBERNETES ##

Containers are *stateless* by design, which means that data does not persist in the containers. Even when you run the containers in kubernetes pods, they still remain
stateless unless you ensure that your configuration supports statefulness.

To achieve statefuleness in kubernetes, you must understand how *volumes, persistent volumes, and persistent volume claims* work.

### Volumes ###
On-disk files in a container are ephemeral, which presents some problems for non-trivial applications when running in containers. One problem is the loss of files when
a container crashes. The kubelet restarts the container but with a clean state. A second problem occurs when sharing files between containers running together in a Pod. 
The Kubernetes volume abstraction solves both of these problems

Docker has a concept of volumes, though it is somewhat looser and less managed. A Docker volume is a directory on disk or in another container. Docker provides volume
drivers, but the functionality is somewhat limited.

Kubernetes supports many types of volumes. A Pod can use any number of volume types simultaneously. Ephemeral volume types have a lifetime of a pod, but persistent
volumes exist beyond the lifetime of a pod. When a pod ceases to exist, Kubernetes destroys ephemeral volumes; however, Kubernetes does not destroy persistent volumes. 
For any kind of volume in a given pod, data is preserved across container restarts.

At its core, a volume is a directory, possibly with some data in it, which is accessible to the containers in a pod. How that directory comes to be, the medium that
backs it, and the contents of it are all determined by the particular volume type used. This means, you must know some of the different types of volumes available in 
kubernetes before choosing what is ideal for your particular use case.

**awsElasticBlockStore**

An awsElasticBlockStore volume mounts an Amazon Web Services (AWS) EBS volume into the pod. The contents of an EBS volume are persisted and the volume is only 
unmmounted when the pod crashes, or terminates. This means that an EBS volume can be pre-populated with data, and that data can be shared between pods.

This exercise (using nignx pod) shows what using *awsElasticBlockStore* volume looks like when used as a PV (Persistent Volume):

Apply this K8s Deployment manifest
~~~
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-volume
        # This AWS EBS volume must already exist.
        awsElasticBlockStore:
          volumeID: "<volume id>"
          fsType: ext4
EOF
~~~

The **Volumes** section indicates the type of volume to be used to ensure persistence.

If you notice the config above carefully, you will realise that there is need to provide a **volumeID** before the deployment will work. Therefore, 
You must create an **EBS** volume by using aws ec2 create-volume command or the AWS console.

Before we create a volume, lets run the nginx deployment into kubernetes without a volume.

~~~
sudo cat <<EOF | sudo tee ./nginx-pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF
~~~

**Tasks**

- Verify that the pod is running
- Check the logs of the pod
- Exec into the pod and navigate to the nginx configuration file /etc/nginx/conf.d
- Open the config files to see the default configuration.

**NOTE:** There are some restrictions when using an awsElasticBlockStore volume:

- The nodes on which pods are running must be AWS EC2 instances
- Those instances need to be in the same region and availability zone as the EBS volume
- EBS only supports a single EC2 instance mounting a volume
