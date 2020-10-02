---
title: "Building a Docker image on GCP"
teaching: 10
exercises: 30
questions:
- "How do I push to a private registry on GCP?"
objectives:
- "Build a Docker image on GCP and push it to the private container registry GCR."
keypoints:
- "GCP allows you to store your Docker images in your own private container registry."
---

## Creating your Docker image

The first thing to do when you want to create your Dockerfile is to ask yourself what you want to do with it. Our goal here is to run a physics analysis with the ROOT framework. To do this our Docker container must contain ROOT as well as all the necessary dependencies it requires. Fortunately there are already existing images matching this, such as `rootproject/root-conda:6.18.04`. We will use this image as our base.

Next we need our source code. We will `git clone` this into our container from the Github repository [https://github.com/awesome-workshop/payload.git](https://github.com/awesome-workshop/payload.git).

To save time we will compile the source code. The other option is to add the source code to our image without compiling it. However by compiling the code at this step, we avoid having to do it over and over again as our workflow runs multiple replicas of our container.

Create a file `compile.sh` with the following contents

```bash
#!/bin/bash

# Compile executable
echo ">>> Compile skimming executable ..."
COMPILER=$(root-config --cxx)
FLAGS=$(root-config --cflags --libs)
$COMPILER -g -O3 -Wall -Wextra -Wpedantic -o skim skim.cxx $FLAGS
```

Finally ceate the file `Dockerfile` with the following contents

```yaml
# The Dockerfile defines the image's environment
# Import ROOT runtime
FROM rootproject/root-conda:6.18.04

# Argument available during build-time
ARG HOME=/home

# Set up working directory
WORKDIR $HOME

# Clone source code
RUN git clone https://github.com/awesome-workshop/payload.git
WORKDIR $HOME/payload

# Compile source code
ADD compile.sh $HOME/payload
RUN ./compile.sh
```

## Building your Docker image

Building your Docker image on GCP is done in the same was as you were showed earlier during the workshop.

```shell
docker build -t [IMAGE-NAME]:[TAG] .
```

If you do not specify a container registry you will push your image to DockerHub, the default registry. We want to push our images to the GCP container registry. To do so we need to name it

```shell
 gcr.io/[PROJECT-ID]/[IMAGE-NAME]:[TAG]
```

where `[PROJECT-ID]` is your GCP project ID. You can find your ID by clicking your project in the top left corner. In our case the ID is `cern-cms`.  The registry hostname is `gcr.io`. `[IMAGE-NAME]` and `[TAG]` are up to you to choose, however rule of thumb is to be as descriptive as possible. Due to Docker naming rules we aslo have to keep them lowercase. One example of how the command could look like is this. 

```shell
docker build -t gcr.io/cern-cms/cms-gXXX:higgstautau .
```
Replace `XXX` with the number for the login credentials you received.

> ## Choose a unique name
>
>Note that all attendants of this workshop are sharing the same registry! Hence you need to try to choose a
unique name and tag combination.
{: .callout}

## Adding your image to the container registry

Before you push or pull images on GCP you need to configure Docker to use the `gcloud` command-line tool to autheticate with the registry. Run the following command

```shell
gcloud auth configure-docker
```

To push the image run

```shell
docker push gcr.io/cern-cms/cms-gXXX:higgstautau
```

> ## View your images
>
> You can view images hosted by the container registry via the cloud console, or by visiting the image's registry name in your web browser at `http://gcr.io/cern-cms/cms-gXXX`.
{: .callout}

## Cleaning up your private registry

To avoid incurring charges to your GCP account you can remove your Docker image from the container registry. Run the follwoing command

```shell
gcloud container images delete gcr.io/cern-cms/cms-gXXX:higgstautau --force-delete-tags
```