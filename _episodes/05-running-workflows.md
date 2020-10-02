---
title: "Running large-scale workflows"
teaching: 10
exercises: 10
questions:
- "How can I run more than a toy workflow?"
objectives:
- "Run a workflow with several parallel jobs"
keypoints:
- "Argo Workflows on Kubernetes are very powerful."
---


## Autoscaling

Kubernetes supports autoscaling to optimise your nodes’ resources as well as adjust CPU and memory to meet your application’s real usage. The Google Kubernetes engine autoscaler automatically resizes the number of nodes in a given node pool, based on the demands of your workloads. 

To configure the autoscaler you simply specify a minimum and maximum for the node pool. The autoscaler then periodically checks the status of the pods and takes acton accordingly.

## Deleting workflows automatically

## Scaling down

Occasionally, the cluster autoscaler cannot scale down completely and extra nodes are left hanging behind. Some sithuations like those can be found documented [here]](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#i-have-a-couple-of-nodes-with-low-utilization-but-they-are-not-scaled-down-why). Therefore it is useful to know how to manually scale down your cluster.

Click on your cluster, listed at `Kubernetes Engine - Clusters`. Scroll down to the end of the page where you will find the `Node pools` section. Clicking on your node pool will take you to its details page.

<kbd>
<img src="../fig/downscale-nodepools.png">
</kbd>

In the upper right corner, next to `EDIT` and `DELETE` you'll find `RESIZE`. 

<kbd>
<img src="../fig/downscale-resize.png">
</kbd>

Clicking on `RESIZE` opens a textfield that allows you to manually adjust the number of pods in your cluster.