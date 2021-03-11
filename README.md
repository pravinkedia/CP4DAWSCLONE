# CP4DAWSCLONE

Cloning & Reinstate between 2 AWS CP4D instances

Cloning from the Source OCP/CP4D cluster and Reinstate to new OCP cluster has the following steps.

## New Developments and Enhancements

### 11-03-2021
1. We now have option to pass this as a login/Token or OC user/password options in the clone and reinstate tool.
2. The logs are now stored on S3

## CLONE

### Pre-requisites for Cloning
Pre-requisites before running cloner utility.

1.	S3 /ICOS where AIP user can create a bucket and save the clones to
2.	OCP 4.5.6 or higher cluster with OCP/CP4D/DV/DMC services installed.
3.	Run the oc login for the existing OCP/CP4D cluster.
4.	Install podman/docker/jq utility if not already exists on the bastion node.
5.	From the oc login you will get the SERVER and TOKEN parameters we need.
6.	OR we have the option now to pass the user and password to the tool.
7.	Install the docker on the machine host from which you are going to run the commands. This can be a bastion node in the AIP AWS environment or any node from which both the source and target OCP/CP4D clusters are accessible.
8.	Ensure the cpd-install-operator on AWS source cluster is up and running.
9.	Ensure all the pod services are in healthy state before starting the clone.
10.	All other DV, DMC pods are also running.

### Always use the latest script docker images as they have latest scripts.

docker pull quay.io/drangar_us/cpc:cp4d.3.5
docker pull quay.io/drangar_us/sincr:v2

#### If in doubt you can delete and pull again

##### To check the docker images
docker image -a

##### Remove the old image and pull again always if in doubt.

docker rm <container id>
docker rm <container id> --force (If needed)
sudo podman rmi <container image id> --force (if needed)

### Cloning from Existing cluster

To run the cloner, you will use the following syntax (You can use docker or podman)

docker run -e ACTION=CLONE -e CLEANUP=1 -e INTERNALREGISTRY=1 -e SERVER=<OC LOGIN SERVER> -e TOKEN=<OC LOGIN TOKEN> -e SINCRIMAGE=quay.io/drangar_us/sincr:v2 -e PROJECT=<OC PROJECT NAME> -e COSBUCKET=<S3 BUCKET NAME> -e COSAPIKEY=<S3 ACCESS KEY> -e COSSECRETKEY=<S3 SECRET KEY> -e COSREGION=<S3 REGION> -e COSENDPOINT=<S3 END POINT> -it quay.io/drangar_us/cpc:cp4d.3.5 

E.G. Command
-

  podman run -e ACTION=CLONE -e INTERNALREGISTRY=0 -e CLEANUP=1 -e SERVER=https://api.drtest2.cp.fyre.ibm.com:6443 -e OCUSER=kubeadmin -e OCPASSWORD=XXXXXXXXXXXXXXXXXX -e SINCRIMAGE=quay.io/drangar_us/sincr:v2 -e PROJECT=streams -e COSBUCKET=mar9drdemoc -e COSAPIKEY=AAAAAAAAAAAA -e COSSECRETKEY=BBBBBBBBBBBBB -e COSREGION=us-south -e COSENDPOINT=https://s3.us-south.cloud-object-storage.appdomain.cloud -it quay.io/drangar_us/cpc:cp4d.3.5

OR

E.G. Command
-
  bash-3.2$ cat cloneawsbackup_new.sh 
  
  sudo podman run -e ACTION=CLONE -e CLEANUP=1 -e INTERNALREGISTRY=1 -e SERVER=https://api.ocp-dv-03.osonawsonline.com:6443 -e TOKEN=sha256~0o-C8TtctoYSWv459O6QYD7s0KQU0bFuTX1fveHPjVo -e SINCRIMAGE=quay.io/drangar_us/sincr:v2 -e PROJECT=cpd-tenant -e COSBUCKET=aipawsbackuppravin -e COSAPIKEY=b737054a3ed54cf0812ddbd56c5efe8d -e COSSECRETKEY=55a9a6c6d7fc9e70c4038c54818b88b3f3fed52874cd47e9 -e COSREGION=au-syd -e COSENDPOINT=https://s3.au-syd.cloud-object-storage.appdomain.cloud -it quay.io/drangar_us/cpc:cp4d.3.5

### To check the backups on the S3 /ICOS storage start with clean bucket.

aws --endpoint-url=https://s3.au-syd.cloud-object-storage.appdomain.cloud s3 ls s3://aipawsbackuppravin/

aws --endpoint-url=https://s3.au-syd.cloud-object-storage.appdomain.cloud s3 ls –summarize s3://aipawsbackuppravin/

## REINSTATE

### Pre-requisites for Reinstate.

Pre-requisites before running the reinstate utility.

1.	Ensure you are running this from the same bastion node where you ran the clone command earlier and it has access to the new cluster.
2.	If using external registry, the target OCP cluster would already have been configured to use the external OpenShift registry.
3.	If using internal registry then you will need to follow the procedure below
4.	You first need to pull/download the images matching with the source cluster images to bastion node for 261 or 263 patches.
5.	Install the 261 or 263-patch using the procedure below.
6.	Then load these images to the targe cluster from bastion node.
7.	Download the matching version of cpd-cli to this bastion node from the following link if not already done https://github.com/IBM/cpd-cli/releases

### Install the Patch 261/263 using the procedure shared earlier.

1.	Use the steps 1-4 a only (Patch 261)
2.	Use the steps 1-4 a only (Patch 263)
 
### If using External Registry (Currently Terraform scripts are being enhanced)

In this approach the target cluster already has access to registry to download the images directly from this internet registry and hence no need to download.

If using internal Registry, we reinstate with INTERNALREGISTRY=1 (pre-load images) otherwise INTERNALREGISTRY=0

### Reinstate for new OpenShift cluster with OCP installed. 

1.	Assume that the new cluster has CP4D 3.5.2 installed with DV, DMC etc.
2.	Then the reinstate command with load the images from either internal or external registry.
3.	Install podman/docker/jq utility if not already exists on the bastion node.
4.	If cpd-meta-ops project does not exist create it manually before starting the reinstate (oc create project cpd-meta-ops) in the target cluster.
5.	Ensure no project cpd-tenant exists in target cluster.
6.	You can l issue the re-instate command in the new cluster by provide the new cluster details using either the docker or podman.

docker run -e REGISTRYPROJECT=<cpd-meta-ops> -e ICR_KEY=<KEY> -e TARGET_REGISTRY==$(oc get route -n openshift-image-registry | grep image-registry | awk '{print $2}') -e TARGET_REGISTRY_USER=$(oc whoami) -e TARGET_REGISTRY_PASSWORD=$(oc whoami -t) -e CLEANUP=0/1 -e ACTION=REINSTATE -e INTERNALREGISTRY=1 -e SERVER=<OC LOGIN SERVER> -e TOKEN=<OC LOGIN TOKEN> -e SINCRIMAGE=quay.io/drangar_us/sincr:v2 -e PROJECT=<OC PROJECT NAME> -e COSBUCKET=<S3 BUCKET NAME> -e COSAPIKEY=<S3 ACCESS KEY> -e COSSECRETKEY=<S3 SECRET KEY> -e COSREGION=<S3 REGION> -e COSENDPOINT=<S3 END POINT> -it quay.io/drangar_us/cpc:cp4d.3.5 

E.G. Command

bash-3.2$ cat cloneawsreinstate_new.sh 

sudo podman run -e REGISTRYPROJECT=cpd-meta-ops -e CLEANUP=0 -e ICR_KEY=eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJJQk0gTWFya2V0cGxhY2UiLCJpYXQiOjE2MTM1NjQxMTUsImp0aSI6ImFhYmI4NDNhYmE3NDQ4NGZhNzhiYzRlODJjOWRmMjQzIn0.2KVoSRLrAY522wXJiH0fKIVXohV-ZxECXDHhry9IuIo -e TARGET_REGISTRY=$(oc get route -n openshift-image-registry | grep image-registry | awk '{print $2}') -e TARGET_REGISTRY_USER=$(oc whoami) -e TARGET_REGISTRY_PASSWORD=$(oc whoami -t) -e ACTION=REINSTATE -e INTERNALREGISTRY=1 -e SERVER=https://api.ocp-dv-02.osonawsonline.com:6443 -e TOKEN=sha256~MSHyLqzxdIcfC1M7qx0utUn_Qsnbzib92EZ3cavF-3U -e SINCRIMAGE=quay.io/drangar_us/sincr:v2 -e PROJECT=cpd-tenant -e COSBUCKET=aipawsbackuppravin -e COSAPIKEY=b737054a3ed54cf0812ddbd56c5efe8d -e COSSECRETKEY=55a9a6c6d7fc9e70c4038c54818b88b3f3fed52874cd47e9 -e COSREGION=au-syd -e COSENDPOINT=https://s3.au-syd.cloud-object-storage.appdomain.cloud -it quay.io/drangar_us/cpc:cp4d.3.5

7.	If any Accenture specific patches are used, then those patches need to be applied to the cluster as described in the above section (Patch 263 or 261) 

8.	Validate cluster health.

oc get pods -A | grep -v Run | grep -v Completed

Check all pods are in running or completed state.

9.	If some pods are down, then only below steps needed. (mostly for elastic search pods). Follow the below instructions to fix it.

a)	Scale down the elastic search pods
oc scale sts elasticsearch-master --replicas=0

b)	Remove the finalizer for pvc
oc edit pv $(oc get pv|grep elasticsearch-master-backups|awk '{print $1}')

NOW REMOVE THE FOLLOWING 2 LINES AND SAVE CHANGES
  finalizers:
  - kubernetes.io/pv-protection

c)	Delete the pvc pod so that it will be created properly.

oc delete pv $(oc get pv|grep elasticsearch-master-backups|awk '{print $1}') --force --grace-period=0

d)	Backup the elastic search pod 

oc get pvc elasticsearch-master-backups -n cpd-tenant -o json | jq ’del(.metadata.annotations)’|jq ’del(.metadata.creationTimestamp)’|jq ’del(.status)’|jq ’del(.metadata.finalizers)’|jq ’del(.metadata.managedFields)’|jq ’del(.spec.volumeName)’|jq ’del(.metadata.manager)’|jq ’del(.metadata.resourceVersion)’|jq ’del(.metadata.selfLink)’|jq ‘del(.metadata.uid)’ > elasticsearch-master-backups.json

e)	Edit/Change elastic search pod to not use the pvc

oc edit pvc elasticsearch-master-backups

NOW REMOVE THE FOLLOWING 2 LINES AND SAVE CHANGES

  finalizers:
  - kubernetes.io/pv-protection

f)	Delete the elastic search pod.

oc delete -n cpd-tenant -f elasticsearch-master-backups.json

g)	Create the elastic search pod

oc create -n cpd-tenant -f elasticsearch-master-backups.json

h)	Scale up the elastic search pods.

oc scale sts elasticsearch-master --replicas=3

i)	Validate the cluster health.

oc get pods -A | grep -v Run | grep -v Completed

10.	Validate target cluster health.

oc exec -i dv-engine-0 -n <NAMESPCE> -c dv-engine -- bash -c "/opt/dv/current/liveness.sh --verbose"

11.	Validate queries against DV tables and caches from source clusters work without issues.

12.	There are 2 routes returned from:  oc get routes -n cpd-tenant

13.	Use the cp4d-route

NAME         HOST/PORT                                             PATH   SERVICES        PORT                   TERMINATION            WILDCARD

cp4d-route   cp4d-route-zzz.apps.ocp452-px255-fips07.cp.fyre.ibm.com   ibm-nginx-svc   ibm-nginx-https-port   passthrough            

zzz-cpd         zzz-cpd-zzz.apps.ocp452-px255-fips07.cp.fyre.ibm.com         ibm-nginx-svc   ibm-nginx-https-port   passthrough/Redirect

14.	Following errors can be neglected

![alt text](https://github.com/pravinkedia/CP4DAWSCLONE/blob/main/Errors_to_neglect.png?raw=true)

(https://github.com/pravinkedia/CP4DAWSCLONE/blob/main/Errors_to_neglect.png)

### Reinstate for new OpenShift cluster with OCP and CP4D pre-installed.

1.	We assume that the cpd-meta-ops already exists with operator pods.
2.	If you already have the OCP cluster with CP4D pre-installed, then you could choose to reinstate the cluster in the new namespace cpd-tenant-new.
3.	Or you can drop the existing cpd-tenant namespace and reinstate the cluster into the same namespace. 
4.	You may need to wait for 10 minutes for this drop to happen correctly.
5.	Follow the similar commands as above.
