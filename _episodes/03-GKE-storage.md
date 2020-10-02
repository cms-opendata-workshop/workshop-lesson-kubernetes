---
title: "Storing workflow output on Google Kubernetes Engine"
teaching: 15
exercises: 15
questions:
- "How can I set up shared storage for my workflows?"
- "How to run a simple job and get the the ouput?"
- "How to run a basic CMS Open Data workflow and get the output files?"
objectives:
- "Understand how to set up shared storage and use it in a workflow"
keypoints:
- "CMS Open Data workflows can be run in a commercial cloud environment using modern tools"
---

One concept that is particularly different with respect to running on a local
computer is storage. Similarly as when mounting directories or volumes in Docker,
one needs to mount available storage into a Pod for it to be able to access it.
Kubernetes provides a large amount of
[storage types](https://kubernetes.io/docs/concepts/storage/).

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

Apply this manifest:

```shell
kubectl apply -n argo -f 003-pv-pvc.yaml
```

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

Now we can use this volume in the workflow definition. Create a workflow definition file `argo-wf-volume.yaml` with the following contents:

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

## Get the output file

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
