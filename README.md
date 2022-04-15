# Tezos Node in Kubernetes

[Tezos](https://tezos.com) is a distributed consensus platform with meta-consensus capability. Tezos not only comes 
to consensus about the state of its ledger, like Bitcoin or Ethereum.
It also attempts to come to consensus about how the protocol and the nodes should adapt and upgrade.

## Introduction

This chart bootstraps a single Tezos node deployment on a [Kubernetes](http://kubernetes.io) cluster using 
the [Helm](https://helm.sh) package manager. Docker image was taken from 
[tezos on Docker hub](https://hub.docker.com/r/tezos/tezos/tags).

## Prerequisites

- Kubernetes 1.8+
- Helm 2.16
- PV provisioner support in the underlying infrastructure
- [gcloud](https://cloud.google.com/sdk/install)

## Creating GKE Kubernetes Cluster

```bash
CLUSTER_INDEX=0
CLUSTER_NAME=tezos-node-${CLUSTER_INDEX} && echo "Cluster name is ${CLUSTER_NAME}"
MASTER_ZONE=us-central1-a

gcloud container clusters create $CLUSTER_NAME \
    --num-nodes 1 \
    --enable-autoscaling --max-nodes=1 --min-nodes=1 \
    --machine-type=n1-standard-2 \
    --cluster-version latest \
    --enable-autorepair \
    --enable-ip-alias \
    --zone=$MASTER_ZONE

gcloud container clusters get-credentials $CLUSTER_NAME --zone=$MASTER_ZONE

kubectl create -f storageclass-ssd.yaml 
helm init
bash patch-tiller.sh
```

## Installing the Chart

To install the chart with the release name `tezos`:

```bash
$ helm install --name tezos charts/tezos
```

Follow the instructions in the output of the command to connect to Tezos RPC.

The command deploys Tezos on the Kubernetes cluster in the default configuration.
The [configuration](#configuration) section lists the parameters that can be configured during installation.

> **Tip**: List all releases using `helm list`

## Uninstalling the Chart

To uninstall/delete the `tezos` deployment:

```bash
$ helm delete tezos --purge
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Configuration

The following table lists the configurable parameters of the Tezos chart and their default values.

Parameter                       | Description                                       | Default
------------------------------- | ------------------------------------------------- | ----------------------------------------------------------
`image.repository`              | Image source repository name                      | `tezos/tezos`
`image.tag`                     | `tezos` release tag.                              | `latest-release`
`image.pullPolicy`              | Image pull policy                                 | `IfNotPresent`
`service.rpcPort`               | RPC port                                          | `8732`
`service.p2pPort`               | P2P port                                          | `9732`
`persistence.enabled`           | Create a volume to store data                     | `true`
`persistence.accessMode`        | ReadWriteOnce or ReadOnly                         | `ReadWriteOnce`
`persistence.size`              | Size of persistent volume claim                   | `200Gi`
`resources`                     | CPU/Memory resource requests/limits               | `{}`
`terminationGracePeriodSeconds` | Wait time before forcefully terminating container | `30`


Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```bash
$ helm install --name tezos -f values.yaml charts/tezos
```

> **Tip**: You can use the default [values.yaml](charts/tezos/values.yaml)

Check a [separate file](ops.md) for more details about additional troubleshooting.

