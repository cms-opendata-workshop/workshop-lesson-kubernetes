---
title: "Downloading data using the cernopendata-client"
teaching: 10
exercises: 15
questions:
- "How can I download data from the Open Data portal?"
- "Should I stream the data from CERN or have them available locally?"
objectives:
- "Understand basic usage of cernopendata-client"
keypoints:
- "It is usually of advantage to have the data where to CPU cores are."
---

The [cernopendata-client](https://github.com/cernopendata/cernopendata-client)
is a tool that has recently become available, which allows you to access
records from the
[CERN Open Data portal](https://opendata.cern.ch/) via the command line.

[cernopendata-client-go](https://github.com/clelange/cernopendata-client-go)
is a light-weight implementation of this with a particular focus on usage "in
the cloud". Additionally, it allows you to download records in parallel.
We will be using `cernopendata-client-go` in the following. Also, mind that
you can also execute these commands on your local computer, but they will also
work in the CloudShell (which is `Linux_x86_64`).

## Getting the tool

You can download the latest release from
[GitHub](https://github.com/clelange/cernopendata-client-go/releases/latest.)
For use on your own computer, download the corresponding binary archive for
your processor architecture and operating system. If you are on MacOS Catalina
or later, you will have to right-click on the extracted binary, hold `CTRL`
when clicking on "Open", and confirm again to open. Afterwards, you should
be able to execute the file from the command line.

The binary will have a different name depending on your operating system and
architecture. Execute it to get help and get more help by using the available
sub-commands:

```shell
./cernopendata-client-go --help
./cernopendata-client-go list --help
./cernopendata-client-go download --help
```

As you can see from the releases page, you can also use docker instead of
downloading the binaries:

```shell
docker run --rm -it clelange/cernopendata-client-go --help
```

## Listing files

When browsing the CERN Open Data portal, you will see from the URL that every
record has a number. For example,
[Analysis of Higgs boson decays to two tau leptons using data and simulation of events at the CMS detector from 2012 ](http://opendata.web.cern.ch/record/12350)
has the URL
`http://opendata.web.cern.ch/record/12350`. The record ID is therefore 12350.

You can list the associated files as follows:

```shell
docker run --rm -it clelange/cernopendata-client-go list -r 12350
```

This yields:

```output
http://opendata.cern.ch/eos/opendata/cms/software/HiggsTauTauNanoAODOutreachAnalysis/HiggsTauTauNanoAODOutreachAnalysis-1.0.zip
http://opendata.cern.ch/eos/opendata/cms/software/HiggsTauTauNanoAODOutreachAnalysis/histograms.py
http://opendata.cern.ch/eos/opendata/cms/software/HiggsTauTauNanoAODOutreachAnalysis/plot.py
http://opendata.cern.ch/eos/opendata/cms/software/HiggsTauTauNanoAODOutreachAnalysis/skim.cxx
```

## Downloading files

You can download full records in an analoguous way:

```shell
docker run --rm -it clelange/cernopendata-client-go download -r 12350
```

By default, these files will end up in a directory called `download`, which
contains directories of the record ID (i.e. `./download/12350` here).

## Creating download jobs

As mentioned in the introductory slides, it is advantageous to download the
desired data sets/records to cloud storage before processing them.
With the `cernopendata-client-go` this can be achieved by creating Kubernetes
[Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/).

A job to download a record making use of the volume we have available could
look like this:

```yaml
# job-data-download.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-data-download
spec:
  backoffLimit: 2
  ttlSecondsAfterFinished: 100
  template:
    spec:
      volumes:
        - name: volume-opendata
          persistentVolumeClaim:
            claimName: nfs-<NUMBER>
      containers:
        - name: pod-data-download
          image: clelange/cernopendata-client-go
          args: ["download", "-r", "12351", "-o", "/opendata"]
          volumeMounts:
            - mountPath: "/opendata/"
              name: volume-opendata
      restartPolicy: Never
```

Again, please adjust the `<NUMBER>`.

You can create (submit) the job like this:

```shell
kubectl apply -n argo -f job-data-download.yaml
```

The job will create a pod, and you can monitor the progress both via the pod
and the job resources.

> ## Challenge: Download all records needed
>
> In the following, we will run the
> [Analysis of Higgs boson decays to two tau leptons using data and simulation of events at the CMS detector from 2012 ](http://opendata.web.cern.ch/record/12350),
> for which 9 different records are listed on the webpage. We only downloaded
> the [GluGluToHToTauTau dataset](http://opendata.web.cern.ch/record/12351)
> so far.
> Can you download the remaining ones using `Jobs`?
>
{: .challenge}

> ## Solution: Download all records needed
>
> In principle, the only thing that needs to be changed is the record ID
> argument. However, resources need to have a unique name, which means that
> the job name needs to be changed or the finished job(s) have to be deleted.
>
{: .solution}
