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
curl -LO https://raw.githubusercontent.com/cms-opendata-workshop/workshop-payload-kubernetes/master/001-nfs-server.yaml
```

The file looks like this:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nfs-server
spec:
  replicas: 1
  selector:
    matchLabels:
      role: nfs-server-<NUMBER>
  template:
    metadata:
      labels:
        role: nfs-server-<NUMBER>
    spec:
      containers:
      - name: nfs-server-<NUMBER>
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
            name: mypvc
      volumes:
        - name: mypvc
          gcePersistentDisk:
            pdName: gce-nfs-disk-<NUMBER>
            fsType: ext4
```

Replace all occurences of `<NUMBER>` by your account number, e.g. `023`.
You can edit files directly in the console or by opening the built-in
graphical editor.
Then apply the manifest:

```shell
kubectl apply -n argo -f 001-nfs-server.yaml
```

Then on to the server service:

```shell
curl -OL https://raw.githubusercontent.com/cms-opendata-workshop/workshop-payload-kubernetes/master/002-nfs-server-service.yaml
```

This looks like this:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nfs-server-<NUMBER>
spec:
  ports:
    - name: nfs
      port: 2049
    - name: mountd
      port: 20048
    - name: rpcbind
      port: 111
  selector:
    role: nfs-server-<NUMBER>
```

As above, replace all occurences of `<NUMBER>` by your account number, e.g. `023`
and apply the manifest:

```shell
kubectl apply -n argo -f 002-nfs-server-service.yaml
```

Download the manifest needed to create the _PersistentVolume_ and
the _PersistentVolumeClaim_:

```shell
curl -OL https://raw.githubusercontent.com/cms-opendata-workshop/workshop-payload-kubernetes/master/003-pv-pvc.yaml
```

The file looks as follows:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: <Add IP here>
    path: "/"

---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nfs-<NUMBER>
  namespace: argo
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 100Gi
```

In the line containing `server:` replace `<Add IP here>` by the output
of the following command:

```shell
kubectl get -n argo svc nfs-server-<NUMBER> |grep ClusterIP | awk '{ print $3; }'
```

This command queries the `nfs-server` service that we created above
and then filters out the `ClusterIP` that we need to connect to the
NFS server. Replace `<NUMBER>` as before. Also, adjust the `<NUMBER>`.

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

With the storage pod, you can copy files between the storage element and the
CloudConsole. However, a practical use case would be to run the "big data"
workloads in the cloud, and then download the output to your local desktop or
laptop for further processing. An easy way of making your files available to
the outside world is to deploy a webserver that mounts the storage volume.

We first patch the config of the webserver to be created as follows:

```shell
mkdir conf.d
cd conf.d
curl -sLO https://raw.githubusercontent.com/cms-opendata-workshop/workshop-payload-kubernetes/master/conf.d/nginx-basic.conf
cd ..
kubectl create configmap basic-config --from-file=conf.d -n argo
```

Then prepare to deploy the fileserver by downloading the manifest:

```shell
curl -sLO https://github.com/cms-opendata-workshop/workshop-payload-kubernetes/raw/master/deployment-http-fileserver.yaml
```

Open this file and again adjust the `<NUMBER>`:

```yaml
# deployment-http-fileserver.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    service: http-fileserver
  name: http-fileserver
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      labels:
        service: http-fileserver
    spec:
      volumes:
      - name: volume-output
        persistentVolumeClaim:
          claimName: nfs-<NUMBER>
      - name: basic-config
        configMap:
          name: basic-config
      containers:
      - name: file-storage-container
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
          - mountPath: "/usr/share/nginx/html"
            name: volume-output
          - name: basic-config
            mountPath: /etc/nginx/conf.d
```

Apply and expose the port as a `LoadBalancer`:

```shell
kubectl create -n argo -f deployment-http-fileserver.yaml
kubectl expose deployment http-fileserver -n argo --type LoadBalancer --port 80 --target-port 80
kubectl get pods -n argo
```

Exposing the deployment will take a few minutes. Run the following command to
follow its status:

```shell
kubectl get svc -n argo
```

You will initially see a line like this:

```output
NAME                          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
http-fileserver               LoadBalancer   10.8.7.24    <pending>     80:30539/TCP   5s
```

The `<pending>` `EXTERNAL-IP` will update after a few minutes (run the command
again to check). Once it's there, copy the IP and paste it into a new browser
tab. This should welcome you with a "Hello from NFS" message. In order to
enable file browsing, we need to delete the `index.html` file in the pod.
Determine the pod name using the first command listed below and adjust the
second command accordingly.

```shell
kubectl get pods -n argo
kubectl exec http-fileserver-XXXXXXXX-YYYYY -n argo -- rm /usr/share/nginx/html/index.html
```

> ## Warning: anyone can now access these files
>
> This IP is now accessible from anywhere in the world, and therefore also
> your files (mind: there are charges for outgoing bandwidth). Please delete
> the service again once you have finished downloading your files.
>
> ~~~
> kubectl delete svc/http-fileserver -n argo
> ~~~
> {: .bash}
>
> Run the `kubectl expose deployment` command to expose it again.
>
{: .testimonial}

{% include links.md %}
