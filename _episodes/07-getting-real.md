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

Download the workflow with the following command:

```shell
curl -OL https://raw.githubusercontent.com/cms-opendata-workshop/workshop-payload-kubernetes/master/higgs-tau-tau-workflow.yaml
```
The file will look something like this:

<script src="https://emgithub.com/embed.js?target=https%3A%2F%2Fgithub.com%2Fcms-opendata-workshop%2Fworkshop-payload-kubernetes%2Fblob%2Fmaster%2Fhiggs-tau-tau-workflow.yaml&style=github&showBorder=on&showLineNumbers=on&showFileMeta=on"></script>

Adjust the workflow as follows:

- Replace `gcr.io/cern-cms/root-conda-002:higgstautau` in all places with the name of the image you created
- Adjust `claimName: nfs-<NUMBER>`

Then execute the workflow and keep your thumbs pressed:

```shell
argo submit -n argo --watch higgs-tau-tau-workflow.yaml
```

Good luck!
