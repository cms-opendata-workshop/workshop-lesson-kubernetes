---
title: "Getting started with Google Kubernetes Engine"
teaching: 15
exercises: 15
questions:
- "How to create a cluster on Google Cloud Platform?"
- "How to setup Google Kubernetes Engine?"
- "How to setup a worflow engine to submit jobs?"
- "How to run a simple job and get the the ouput?"
- "How to run a basic CMS Open Data workflow and get the output files?"
objectives:
- "Understand how to run a simple CMS Open Data workflow in a commercial cloud environment"
keypoints:
- "CMS Open Data workflows can be run in a commercial cloud environment using modern tools"

---

## Get access to Google Cloud Platform

Usually, you would need to create your own account (and add a credit card) in
order to be able to access the public cloud. For this workshop, however, you
will have access to Google Cloud via the
[ARCHIVER project](https://www.archiver-project.eu/).
Alternatively, you can create a Google Cloud account and you will get free
credits worth $300, valid for 90 days (requires a credit card).

The following steps will be performed using the
[Google Cloud Platform console](https://console.cloud.google.com/).

> ## Use an incognito/private browser window
>
> We strongly recommend you use an incognito/private browser window
> to avoid possible confusion and mixups with personal Google accounts
> you might have.
>
{: .callout}

## Find your cluster

By now, you should already have created your first Kubernetes cluster.
In case you do not find it, use the "Search products and resources" field
to search and select "Kubernetes Engine". It should show up there.

## Open the working environment

You can work from the cloud shell, which opens from an icon in the tool bar,
indicated by the red arrow in the figure below:

<kbd>
<img src="../fig/startcloudshell.png">
</kbd>

In the following, all the commands are typed in that shell.
We will make use of many `kubectl` commands. If interested, you can read
[an overview](https://kubernetes.io/docs/reference/kubectl/overview/)
of this command line tool for Kubernetes.

## Install argo as a workflow engine

While jobs can also be run manually, a workflow engine makes defining and
submitting jobs easier. In this tutorial, we use
[argo](https://argoproj.github.io/argo/quick-start/).
Install it into your working environment with the following commands
(all commands to be entered into the cloud shell):

```bash
kubectl create ns argo
kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo/stable/manifests/quick-start-postgres.yaml
curl -sLO https://github.com/argoproj/argo/releases/download/v2.11.1/argo-linux-amd64
chmod +x argo-linux-amd64
sudo mv ./argo-linux-amd64 /usr/local/bin/argo
```

This will also install the argo binary, which makes managing the workflows
easier.

You need to execute the following command so that the argo workflow controller
has sufficient rights to manage the workflow pods.
Replace `XXX` with the number for the login credentials you received.

```bash
kubectl create clusterrolebinding cern-cms-cluster-admin-binding --clusterrole=cluster-admin --user=cms-gXXX@arkivum.com
```

You can now check that argo is available with

```bash
argo version
```

We need to apply a small patch to the default argo config. Create a file called
`patch-workflow-controller-configmap.yaml`:

```yaml
data:
  artifactRepository: |
    archiveLogs: false
```

Apply:

```shell
kubectl patch configmap workflow-controller-configmap -n argo --patch "$(cat patch-workflow-controller-configmap.yaml)"
```

## Run a simple test workflow

To test the setup, run a simple test workflow with

```bash
argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo/master/examples/hello-world.yaml
argo list -n argo
argo get -n argo @latest
argo logs -n argo @latest
argo delete -n argo @latest
```

Please mind that it is important to delete your workflows once they have
completed. If you do not do this, the pods associated with the workflow
will remain scheduled in the cluster, which might lead to additional charges.
You will learn how to automatically remove them later.

> ## Kubernetes namespaces
>
> The above commands as well as most of the following use a flag `-n argo`,
> which defines the namespace in which the resources are queried or created.
> [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
> separate resources in the cluster, effectively giving you multiple virtual
> clusters within a cluster.
>
> You can change the default namespace to `argo` as follows:
>
> ~~~
> kubectl config set-context --current --namespace=argo
> ~~~
> {: .bash}
>
{: .testimonial}

## Defining a storage volume

The test job above did not produce any output data files, just text logs.
The data analysis jobs will produce output files and, in the following, we will
go through a few steps to setup a volume where to which the output files will be
written and from where they can be fetched.
All definitions are passed as "yaml" files, which you've already used in the
steps above. Due to some restrictions of the Google Kubernetes Engine, we need
to use an NFS persistent volumes to allow parallel access (see also
[this post](https://medium.com/platformer-blog/nfs-persistent-volumes-with-kubernetes-a-case-study-ce1ed6e2c266)).

The following commands will take care of this. You can look at the content of
the files that are directly applied from GitHub at the
[workshop-paylod-kubernetes](https://github.com/cms-opendata-workshop/workshop-payload-kubernetes)
repository.

```shell
kubectl apply -n argo -f https://raw.githubusercontent.com/cms-opendata-workshop/workshop-payload-kubernetes/master/001-nfs-server.yaml
kubectl apply -n argo -f https://raw.githubusercontent.com/cms-opendata-workshop/workshop-payload-kubernetes/master/002-nfs-server-service.yaml
```

Then download the manifest needed to create the _PersistentVolume_ and
the _PersistentVolumeClaim_:

```shell
curl -OL https://raw.githubusercontent.com/cms-opendata-workshop/workshop-payload-kubernetes/master/003-pv-pvc.yaml
```

In the line containing `server:` replace `<Add IP here>` by the output
of the following command:

```shell
kubectl get svc nfs-server |grep ClusterIP | awk '{ print $3; }'
```

This command queries the `nfs-server` service that we created above
and then filters out the `ClusterIP` that we need to connect to the
NFS server.

You can edit files directly in the console or by opening the built-in
graphical editor.

Let's confirm that this worked:

```bash
kubectl get pvc nfs -n argo
```

You will see output similar to this

```output
NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs    Bound    nfs      10Gi       RWX                           5h2m
```

Note that it may take some time before the STATUS gets to the state "Bound".

Now we can use this volume in the workflow definition. Create a workflow definition file `argo_wf_volume.yaml` with the following contents:

```yaml
# argo-wf-volume.ysml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: test-hostpath-
spec:
  entrypoint: test-hostpath
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: nfs
  templates:
  - name: test-hostpath
    script:
      image: alpine:latest
      command: [sh]
      source: |
        echo "This is the ouput" > /mnt/vol/test.txt
        echo ls -l /mnt/vol: `ls -l /mnt/vol`
      volumeMounts:
      - name: task-pv-storage
        mountPath: /mnt/vol
```

Submit and check this workflow with

```bash
argo submit -n argo argo-wf-volume.yaml
argo list -n argo
```

Take the name of the workflow from the output (replace XXXXX in the following command)
and check the logs:

```bash
kubectl logs pod/test-hostpath-XXXXX  -n argo main
```

Once the job is done, you will see something like:

```output
ls -l /mnt/vol: total 20 drwx------ 2 root root 16384 Sep 22 08:36 lost+found -rw-r--r-- 1 root root 18 Sep 22 08:36 test.txt
```

## Get the ouput file

The example job above produced a text file as an output. It resides in the persistent volume that the workflow job has created.
To copy the file from that volume to the cloud shell, we will define a container, a "storage pod" and mount the volume there so that we can get access to it.

Create a file `pv-pod.yaml` with the following contents:

```yaml
# pv-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: nfs
  containers:
    - name: pv-container
      image: busybox
      command: ["tail", "-f", "/dev/null"]
      volumeMounts:
        - mountPath: /mnt/data
          name: task-pv-storage
```

Create the storage pod and copy the files from there

```bash
kubectl apply -f pv-pod.yaml -n argo
kubectl cp  pv-pod:/mnt/data /tmp/poddata -n argo
```

and you will get the file created by the job in `/tmp/poddata/test.txt`.

## Run a CMS open data workflow

If the steps above are successful, we are now ready to run a workflow to process CMS open data.

Create a workflow file `argo-wf-cms.yaml` with the following content:

```yaml
# argo-wf-cms.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: nanoaod-argo-
spec:
  entrypoint: nanoaod-argo
  volumes:
  - name: task-pv-storage
    persistentVolumeClaim:
      claimName: nfs
  templates:
  - name: nanoaod-argo
    script:
      image: cmsopendata/cmssw_5_3_32
      command: [sh]
      source: |
        source /opt/cms/entrypoint.sh
        sudo chown $USER /mnt/vol
        mkdir workspace
        cd workspace
        git clone git://github.com/cms-opendata-analyses/AOD2NanoAODOutreachTool  AOD2NanoAOD
        cd AOD2NanoAOD
        scram b -j8
        nevents=100
        eventline=$(grep maxEvents configs/data_cfg.py)
        sed -i "s/$eventline/process.maxEvents = cms.untracked.PSet( input = cms.untracked.int32($nevents) )/g" configs/data_cfg.py
        cmsRun configs/data_cfg.py
        cp output.root /mnt/vol/
        echo  ls -l /mnt/vol
        ls -l /mnt/vol
      volumeMounts:
      - name: task-pv-storage
        mountPath: /mnt/vol
```

Submit the job with

```bash
argo submit -n argo argo-wf-cms.yaml --watch
```

The option `--watch` gives a continuous follow-up of the progress. To get the logs of the job, use the process name (`nanoaod-argo-XXXXX`) which you can also find with

```bash
argo get -n argo @latest
```

and follow the container logs with

```bash
kubectl logs pod/nanoaod-argo-XXXXX  -n argo main
```

Get the output file `output.root` from the storage pod in a similar manner as it was done above.

## Accessing files via http



{% include links.md %}
