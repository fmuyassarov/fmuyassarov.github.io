---
title: "Kubernetes Prow with Azure blob storage"
date: "2021-10-09"
tags: ["metal3"]
summary: "How to setup a Prow with Azure blob storage."
ShowToc: true
weight: 5
---

Recently I had to build a [Kubernetes Prow](https://github.com/kubernetes/test-infra/tree/master/prow) CICD system for one of the projects I was working on. Usually you want to store Prow job artifacts in some cloud storage so that they are available for future
use debugging. In my case, Azure cloud was the only option to go with. Unfortunately, at the time of writing this post Prow doesn't support Azure as a storage backed and only GCP or AWS S3.

However, thanks to [MinIO](https://github.com/minio/minio), I could build my Prow CICD cluster and still store the job artifacts in Azure storage as I would in GCP or AWS S3. For that reason, I wanted to share how I configured my Prow cluster to work with Azure storage.

**Note:** This is Prow setup in a local test environment. In other words, there is no TLS, cert-manager, ingress controller configuration involved as they would in real setup.

## Prerequisites

1. Github bot account
2. Github organization
3. Kubernetes cluster
4. Azure Blob Storage Account
5. Azure Blob Storage access and secret key

## Setup
1. I assume you already have a local Kubernetes cluster running. I used [Minikube](https://github.com/kubernetes/minikube) to spin up my cluster.

1. Generate a HMAC token and create a secret from that token. We will also pass it to our GitHub org webhook configuration later on.
    ```shell
    $ openssl rand -hex 20 > /path/hmac-token
    $ kubectl create secret generic hmac-token --from-file=hmac=/path/hmac-token
    ```
1. Create a personal access token from GitHub bot account with the following fields checked in. Instructions for creating an access token is [here](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token).
    - `public_repo` and `repo:status`
    - `repo` scope for private repos
    - `admin:org_hook` for the github org

   Now create a secret from this token
    ```shell
    $ kubectl create secret generic github-token --from-file=token=/path/github-token
    ```
1. I've used upstream [starter-s3.yaml](https://github.com/kubernetes/test-infra/blob/master/config/prow/cluster/starter-s3.yaml) YAML that is good enough to set up all the components of Prow. But I had to make some changes to the file because by default MinIO deployment is not meant to be used as gateway mode in that YAML but rather as server mode with local storage.
    - First I changed MinIO deployment `args` part to be as follow 
        
        ```yaml
        args:
        - gateway
        - azure
        - --console-address=:33333
        ```
       These parameters will instruct the MinIO to run in gateway mode, with Azure being the storage backend. Also, I added            a console address port so that I have a predictable port number that I will use when accessing MinIO web console.

    - Then I removed MinIO `PersistentVolumeClaim` and `volumeMounts` from MinIO deployment because we are not going to use   local volume as it's configured in that YAML. 
    - I also removed the initContainer part from MinIO deployment.
    
    At the end, my MinIO deployment looked like [this](https://gist.github.com/fmuyassarov/d73884ddb13fd903645d2b3e3cf35120).

    ```shell
    $ kubectl apply -f custom_starter-s3.yaml
    ```
1. Since this was a test environment, I used [ultrahook](https://www.ultrahook.com/) to get a temporary public URL.
    ```shell
    $ ultrahook github http://192.168.49.2:31723/hook
    ```
    This generated me a public URL, e.g., https://user-github.ultrahook.com which forwards all recevied events' traffic to http://192.168.49.2:31723/hook. As you probably noticed, we are forwarding all the GitHub events data to hook service running inside the Minikube cluster. 
    - `192.168.49.2` - Minikube IP (*```$ minikube ip```*)
    - `31723` - hook service nodePort


1. At this stage, I configured [webhook](https://docs.github.com/en/rest/reference/orgs#webhooks) of my GitHub organization to send webhook payloads (`send me everything` for the type of the events to be sent) to https://user-github.ultrahook.com. You can find webhook instructions [here](https://docs.github.com/en/github/setting-up-and-managing-your-enterprise/managing-organizations-in-your-enterprise-account/configuring-webhooks-for-organization-events-in-your-enterprise-account).

1. To get access to the web console of MinIO, I created a service of type `nodePort` based on the YAML below. I've configured `targetPort` to be 33333 which is the same port that I passed to MinIO deployment with `--console-address` argument. This argument allows you to specify the MinIO console port instead of getting a random one.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: minio-console
      namespace: prow
    spec:
      type: NodePort
      ports:
      - port: 8003
        targetPort: 33333
        protocol: TCP
      selector:
        app: minio
    ```
    Once `minio-console` service was running, I could open the web console via
    ```shell
    $ minikube service -n prow minio-console
    ```

    ![minio](https://user-images.githubusercontent.com/35802557/132105961-e66f0b04-6e2a-48e5-8f64-72a409d90e95.png)

    And I could see the same data being stored in Azure blob storage

    ![azure](https://user-images.githubusercontent.com/35802557/132965756-15cb7917-ab01-4662-af87-84546c9a217e.png)


**Note:** I mentioned above that I removed `initContainer` part too form the starter-s3.yaml and for that reason I had to create `prow-logs`, `status-reconciler`, `tide` buckets manually from MinIO console. You can also do it from Azure.