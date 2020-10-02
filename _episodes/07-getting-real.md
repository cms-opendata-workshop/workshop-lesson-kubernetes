---
title: "Getting real"
teaching: 5
exercises: 20
questions:
- "How can I now use this for real?"
objectives:
- "Get an idea of a full workflow with argo"
keypoints:
- "Argo is a powerful tool for running parallel workflows"
---

So far we've run smaller examples, but now we have everything at hands to run
physics analysis jobs in parallel.

Create a workflow file `higgs-tau-tau-workflow.yaml` with the following content:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: higgs-tau-tau-
spec:
  arguments:
    parameters:
    - name: files-list
      value: |
        [
          {"file": "GluGluToHToTauTau", "x-section": "19.6", "process": "ggH"},
          {"file": "VBF_HToTauTau", "x-section": "1.55", "process": "ggH"},
          {"file": "DYJetsToLL", "x-section": "3503.7", "process": "ZTT"},
          {"file": "TTbar", "x-section": "255.2", "process": "TT"},
          {"file": "W1JetsToLNu", "x-section": "6381.2", "process": "W1J"},
          {"file": "W2JetsToLNu", "x-section": "2039.8", "process": "W2J"},
          {"file": "W3JetsToLNu", "x-section": "612.5", "process": "W3J"},
          {"file": "Run2012B_TauPlusX", "x-section": "1.0", "process": "dataRunB"},
          {"file": "Run2012C_TauPlusX", "x-section": "1.0", "process": "dataRunC"}
        ]
    - name: histogram-list
      value: |
        [
          {"file": "GluGluToHToTauTau", "x-section": "19.6", "process": "ggH"},
          {"file": "VBF_HToTauTau", "x-section": "1.55", "process": "ggH"},
          {"file": "DYJetsToLL", "x-section": "3503.7", "process": "ZTT"},
          {"file": "DYJetsToLL", "x-section": "3503.7", "process": "ZLL"},
          {"file": "TTbar", "x-section": "255.1", "process": "TT"},
          {"file": "W1JetsToLNu", "x-section": "6381.2", "process": "W1J"},
          {"file": "W2JetsToLNu", "x-section": "2039.8", "process": "W2J"},
          {"file": "W3JetsToLNu", "x-section": "612.5", "process": "W3J"},
          {"file": "Run2012B_TauPlusX", "x-section": "1.0", "process": "dataRunB"},
          {"file": "Run2012C_TauPlusX", "x-section": "1.0", "process": "dataRunC"}
        ]
  entrypoint: parallel-worker
  volumes:
  - name: task-pv-storage
    persistentVolumeClaim:
      claimName: nfs-<NUMBER>
  templates:
  - name: parallel-worker
    inputs:
      parameters:
      - name: files-list
      - name: histogram-list
    dag:
      tasks:

      - name: skim-step
        template: skim-template
        arguments:
          parameters:
          - name: file
            value: "{{item.file}}"
          - name: x-section
            value: "{{item.x-section}}"
        withParam: "{{inputs.parameters.files-list}}"

      - name: histogram-step
        dependencies: [skim-step]
        template: histogram-template
        arguments:
          parameters:
          - name: file
            value: "{{item.file}}"
          - name: process
            value: "{{item.process}}"
        withParam: "{{inputs.parameters.histogram-list}}"

      - name: merge-step
        dependencies: [histogram-step]
        template: merge-template

      - name: plot-step
        dependencies: [merge-step]
        template: plot-template

      - name: fit-step
        dependencies: [merge-step]
        template: fit-template

  - name: skim-template
    inputs:
      parameters:
      - name: file
      - name: x-section
    script:
      image: gcr.io/cern-cms/root-conda-002:higgstautau
      command: [sh]
      source: |
        LUMI=11467.0 # Integrated luminosity of the unscaled dataset
        SCALE=0.1 # Same fraction as used to down-size the analysis
        mkdir -p $HOME/skimm
        ./skim root://eospublic.cern.ch//eos/opendata/cms/derived-data/AOD2NanoAODOutreachTool/{{inputs.parameters.file}}.root $HOME/skimm/{{inputs.parameters.file}}-skimmed.root {{inputs.parameters.x-section}} $LUMI $SCALE
        ls -l $HOME/skimm
        cp $HOME/skimm/* /mnt/vol
      volumeMounts:
      - name: task-pv-storage
        mountPath: /mnt/vol
      resources:
        limits:
          memory: 2Gi
        requests:
          memory: 1.7Gi
          cpu: 750m

  - name: histogram-template
    inputs:
      parameters:
      - name: file
      - name: process
    script:
      image: gcr.io/cern-cms/root-conda-002:higgstautau
      command: [sh]
      source: |
        mkdir -p $HOME/histogram
        python histograms.py /mnt/vol/{{inputs.parameters.file}}-skimmed.root {{inputs.parameters.process}} $HOME/histogram/{{inputs.parameters.file}}-histogram-{{inputs.parameters.process}}.root
        ls -l $HOME/histogram
        cp $HOME/histogram/* /mnt/vol
      volumeMounts:
      - name: task-pv-storage
        mountPath: /mnt/vol
      resources:
        limits:
          memory: 2Gi
        requests:
          memory: 1.7Gi
          cpu: 750m

  - name: merge-template
    script:
      image: gcr.io/cern-cms/root-conda-002:higgstautau
      command: [sh]
      source: |
        hadd -f /mnt/vol/histogram.root /mnt/vol/*-histogram-*.root
        ls -l /mnt/vol
      volumeMounts:
      - name: task-pv-storage
        mountPath: /mnt/vol
      resources:
        limits:
          memory: 2Gi
        requests:
          memory: 1.7Gi
          cpu: 750m

  - name: plot-template
    script:
      image: gcr.io/cern-cms/root-conda-002:higgstautau
      command: [sh]
      source: |
        SCALE=0.1
        python plot.py /mnt/vol/histogram.root /mnt/vol $SCALE
        ls -l /mnt/vol
      volumeMounts:
      - name: task-pv-storage
        mountPath: /mnt/vol

  - name: fit-template
    script:
      image: gcr.io/cern-cms/root-conda-002:higgstautau
      command: [sh]
      source: |
        python fit.py /mnt/vol/histogram.root /mnt/vol
        ls -l /mnt/vol
      volumeMounts:
      - name: task-pv-storage
        mountPath: /mnt/vol
```

Adjust the workflow as follows:

- Replace `gcr.io/cern-cms/root-conda-002:higgstautau` in all places with the name of the image you created
- Adjust `claimName: nfs-<NUMBER>`

Then execute the workflow and keep your thumbs pressed:

```shell
argo submit -n argo --watch higgs-tau-tau-workflow.yaml
```

Good luck!
