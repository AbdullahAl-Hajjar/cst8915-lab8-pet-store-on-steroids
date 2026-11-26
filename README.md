# CST8915 Lab 8: Deploying the Algonquin Pet Store (On Steroids)

**Student Name**: Abdullah Al-Hajjar
**Student ID**: 040656012
**Course**: CST8915 Full-stack Cloud-native Development
**Semester**: Fall 2026

---

## Demo Video

🎥 [Watch Demo Video](https://youtu.be/sQ49r2iStzA)

---

## Files

| File | Purpose |
| --- | --- |
| [`aps-all-in-one-Task1.yaml`](aps-all-in-one-Task1.yaml) | The lab's all-in-one manifest, with the six application services switched to my Docker Hub images (`alha0086/...:lab8`). MongoDB and RabbitMQ still use the official `mongo` and `rabbitmq` images. |
| [`admin-tasks.yaml`](admin-tasks.yaml) | Virtual Customer and Virtual Worker, using my `alha0086/virtual-customer:lab8` and `alha0086/virtual-worker:lab8` images. |
| [`config-maps.yaml`](config-maps.yaml) | The RabbitMQ plugins ConfigMap from the lab (unchanged). |

`secrets.yaml` (the OpenAI API keys) is deliberately **not** in this repository. It's listed in `.gitignore`.

---

## Task 1: Build, Push, and Deploy My Own Docker Images

### My forks and images

I forked all eight service repositories, then built each image and pushed it to my Docker Hub account (`alha0086`):

| Service | My fork | My Docker image |
| --- | --- | --- |
| `store-front` | [AbdullahAl-Hajjar/store-front-L8](https://github.com/AbdullahAl-Hajjar/store-front-L8) | [`alha0086/store-front:lab8`](https://hub.docker.com/r/alha0086/store-front/tags) |
| `store-admin` | [AbdullahAl-Hajjar/store-admin-L8](https://github.com/AbdullahAl-Hajjar/store-admin-L8) | [`alha0086/store-admin:lab8`](https://hub.docker.com/r/alha0086/store-admin/tags) |
| `order-service` | [AbdullahAl-Hajjar/order-service-L8](https://github.com/AbdullahAl-Hajjar/order-service-L8) | [`alha0086/order-service:lab8`](https://hub.docker.com/r/alha0086/order-service/tags) |
| `product-service` | [AbdullahAl-Hajjar/product-service-L8](https://github.com/AbdullahAl-Hajjar/product-service-L8) | [`alha0086/product-service:lab8`](https://hub.docker.com/r/alha0086/product-service/tags) |
| `makeline-service` | [AbdullahAl-Hajjar/makeline-service-L8](https://github.com/AbdullahAl-Hajjar/makeline-service-L8) | [`alha0086/makeline-service:lab8`](https://hub.docker.com/r/alha0086/makeline-service/tags) |
| `ai-service` | [AbdullahAl-Hajjar/ai-service-L8](https://github.com/AbdullahAl-Hajjar/ai-service-L8) | [`alha0086/ai-service:lab8`](https://hub.docker.com/r/alha0086/ai-service/tags) |
| `virtual-customer` | [AbdullahAl-Hajjar/virtual-customer-L8](https://github.com/AbdullahAl-Hajjar/virtual-customer-L8) | [`alha0086/virtual-customer:lab8`](https://hub.docker.com/r/alha0086/virtual-customer/tags) |
| `virtual-worker` | [AbdullahAl-Hajjar/virtual-worker-L8](https://github.com/AbdullahAl-Hajjar/virtual-worker-L8) | [`alha0086/virtual-worker:lab8`](https://hub.docker.com/r/alha0086/virtual-worker/tags) |

### How the images are built and pushed

My PC doesn't have Docker installed, so I built the images in the cloud with **GitHub Actions**. Each fork has a workflow, `.github/workflows/docker-publish.yml`, that runs on every push to `main` (or on demand). It:

1. checks out the code,
2. sets up Docker Buildx,
3. logs in to Docker Hub as `alha0086` with a Docker Hub access token stored as the repository secret `DOCKERHUB_TOKEN`,
4. builds the image from the repository's `Dockerfile` for `linux/amd64` (the AKS nodes' architecture), and
5. pushes it with two tags: `lab8` (used by the manifests) and the commit SHA (so every build can be traced back to its source).

The token is only stored as an encrypted GitHub secret. It isn't in any file or in the workflow.

### Cluster

- **AKS cluster:** `Lab8Cluster` in resource group `CST8915-Lab8`, with two `Standard_B2as_v2` nodes (2 vCPU, 8 GB each). Kubernetes 1.36, Free tier.
- **Region:** North Central US. My Azure for Students subscription only allows a few regions, and Canada is not one of them.
- The AKS metrics server is installed by default, so `kubectl top` works.

### Deployment

From the folder with the manifests:

```bash
kubectl apply -f config-maps.yaml
kubectl apply -f secrets.yaml
kubectl apply -f aps-all-in-one-Task1.yaml
kubectl apply -f admin-tasks.yaml
```

### Verification

After applying both files, every application pod runs one of my images:

```text
$ kubectl get pods -o custom-columns='POD:.metadata.name,STATUS:.status.phase,IMAGE:.spec.containers[0].image'
POD                                 STATUS    IMAGE
ai-service-69fdc8c7c6-q8pr9         Running   alha0086/ai-service:lab8
makeline-service-7b8c98489c-gtqsx   Running   alha0086/makeline-service:lab8
mongodb-0                           Running   mongo:8.0.38
order-service-645f4f576d-rzmjj      Running   alha0086/order-service:lab8
product-service-8d47d7b9-pqcsb      Running   alha0086/product-service:lab8
rabbitmq-0                          Running   rabbitmq:3-management
store-admin-57d897c686-vmnhm        Running   alha0086/store-admin:lab8
store-front-9d7b88d85-mfhkg         Running   alha0086/store-front:lab8
virtual-customer-69b7567f-6wtrx     Running   alha0086/virtual-customer:lab8
virtual-worker-9588c4496-jkd8s      Running   alha0086/virtual-worker:lab8
```

What I tested:

- **Store Front and Store Admin** both load on their LoadBalancer IPs, and the Store Front lists the 10 products from the product service.
- **Virtual Customer** places orders through the order service (`Order 2 sent ... with status of 201 Created`).
- **Virtual Worker** completes them through the makeline service (`Order 67088 completed ... with status 2`).
- **Store Admin** shows the orders that are waiting to be processed.
- **Scaling:** `kubectl scale deployment order-service --replicas=3` ran three `order-service` pods.
- **Metrics:** `kubectl top nodes` and `kubectl top pods` show CPU and memory use. Both nodes were under 10% CPU.
- **RabbitMQ:** through `kubectl port-forward service/rabbitmq 15672:15672`, the management UI and API show RabbitMQ 3.13.7 on node `rabbit@rabbitmq-0` and the `orders` queue.
- **MongoDB:** `kubectl exec -it mongodb-0 -- mongosh` lists the `orderdb` database. Its `orders` collection holds the completed orders (status 2), with customer ID, items and prices.

### Note on the AI service

The lab's Student Guide says the OpenAI credentials come from a `secrets.yaml` file on Brightspace, but no such file was posted for my section. I created `secrets.yaml` from `secrets.example.yaml`, so both Secrets exist with placeholder keys. The `ai-service` pod starts and runs normally, but **Ask AI Assistant** and **Generate Image** in Store Admin can't work until real keys are applied. With a real `secrets.yaml`, I would apply it and run `kubectl rollout restart deployment/ai-service` to reload the keys.

---

## Task 2: Azure Managed Services to Replace MongoDB and RabbitMQ

In `aps-all-in-one.yaml`, MongoDB and RabbitMQ run inside the cluster as single-replica StatefulSets. Neither one has a PersistentVolumeClaim: the orders and the queue live on the container's own filesystem. That means a rescheduled pod loses its data, as I showed for RabbitMQ in Lab 7. Both also have no high availability or backups, and they use weak or no credentials: RabbitMQ is `username`/`password` in plain text, and MongoDB has no authentication at all. Azure has a fully managed service for each.

### Replacing RabbitMQ: Azure Service Bus

**What it is:** Azure Service Bus is a fully managed enterprise message broker with message queues and publish-subscribe topics, used to decouple applications and services (Microsoft, n.d.-a).

**Why it's a good fit:**

- **It speaks the same protocol.** The order service (the `rhea` library) and the makeline service (`go-amqp`) already talk to RabbitMQ over **AMQP 1.0**, which is why `config-maps.yaml` enables RabbitMQ's `rabbitmq_amqp1_0` plugin. Service Bus supports AMQP 1.0 natively (Microsoft, n.d.-b).
- **The code is already there.** Both services include a Service Bus code path. The order service uses `@azure/service-bus`, and the makeline service uses `azservicebus`. Setting `USE_WORKLOAD_IDENTITY_AUTH=true` and the namespace host name (`ORDER_QUEUE_HOSTNAME` or `AZURE_SERVICEBUS_FULLYQUALIFIEDNAMESPACE`) switches them over. So moving to Service Bus is a configuration change, not a rewrite.
- **No passwords.** With Microsoft Entra Workload ID, each pod's service account gets an Azure identity, and Service Bus access is granted with Azure RBAC (Microsoft, n.d.-e). The `username`/`password` values disappear from the manifest.

**What problem it solves:** messages are stored durably by Azure, outside the cluster, so deleting or rescheduling a pod can't lose orders. Service Bus also adds features I would otherwise have to build myself:

- a **dead-letter queue** for orders that can't be processed,
- duplicate detection and sessions,
- geo-replication for regional outages in the Premium tier (Microsoft, n.d.-a).

I wouldn't have to run, patch or scale a RabbitMQ StatefulSet anymore.

**Trade-offs:** Service Bus is a paid service, charged per operation on Standard or per messaging unit on Premium. Private networking needs Premium. The RabbitMQ management UI is replaced by the Azure portal and Service Bus Explorer.

### Replacing MongoDB: Azure DocumentDB (with MongoDB compatibility)

**What it is:** Azure DocumentDB, previously named **vCore-based Azure Cosmos DB for MongoDB**, is a fully managed document database with MongoDB compatibility. Existing clusters, connection strings and APIs didn't change with the new name (Microsoft, n.d.-c).

**Why it's a good fit:**

- **The application doesn't change.** The makeline service uses the standard MongoDB driver with `ORDER_DB_URI=mongodb://mongodb:27017`, so pointing it at a DocumentDB connection string (stored in a Kubernetes Secret) is the only change. Microsoft reports compatibility with 99.03% of MongoDB operators (Microsoft, n.d.-c).
- **Predictable cost.** Pricing is vCore-based, with no extra charge for backups or I/O. A free tier with 32 GB of storage is enough for a store like this (Microsoft, n.d.-c).

**What problem it solves:** the orders get replicated storage, automatic backups, a high-availability SLA and real authentication, instead of a pod with no persistent volume and no password. Scaling becomes a setting, not a StatefulSet change.

**Alternatives in the same family:**

- **Azure Cosmos DB for MongoDB (RU-based)** is also fully managed and speaks the MongoDB wire protocol, so existing drivers and tools work with minimal code changes (Microsoft, n.d.-d).
- The makeline service even has native support for **Azure Cosmos DB for NoSQL** (`ORDER_DB_API=cosmosdbsql`, with keys or Workload ID). That option offers a 99.999% availability SLA, instant autoscale and multi-region failover for a much larger store (Microsoft, n.d.-d).

### Summary

| Self-hosted (in AKS) | Azure managed replacement | Main benefit |
| --- | --- | --- |
| RabbitMQ StatefulSet (no volume) | **Azure Service Bus** (queue `orders`) | Durable, highly available messaging, with dead-lettering and Entra ID authentication. The services already support it. |
| MongoDB StatefulSet (no volume) | **Azure DocumentDB** (or Cosmos DB for MongoDB) | Backed-up, replicated and authenticated database. Only the connection string changes. |

---

## References

Microsoft. (n.d.-a). *Introduction to Azure Service Bus messaging*. Microsoft Learn. https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview

Microsoft. (n.d.-b). *Overview of AMQP 1.0 in Azure Service Bus*. Microsoft Learn. https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-amqp-overview

Microsoft. (n.d.-c). *What is Azure DocumentDB (with MongoDB compatibility)?* Microsoft Learn. https://learn.microsoft.com/en-us/azure/documentdb/overview

Microsoft. (n.d.-d). *Introduction/Overview - Azure Cosmos DB for MongoDB*. Microsoft Learn. https://learn.microsoft.com/en-us/azure/cosmos-db/mongodb/overview

Microsoft. (n.d.-e). *Use a Microsoft Entra Workload ID on Azure Kubernetes Service (AKS)*. Microsoft Learn. https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview
