# Xnode Cortx on Google Cloud Marketplace

Xnode Cortx is an AI control plane for agentic AI that runs inside your own network on Google
Kubernetes Engine (GKE): agent orchestration, MCP tools, guardrails, RBAC and observability.

This repository holds the Google Cloud Marketplace deployment package for Cortx and the steps to
deploy it **from the command line**. You can also deploy from the
[Google Cloud Marketplace listing](https://console.cloud.google.com/marketplace/product/xnode-public/xnode-cortx-ai-control-plane-byol)
in the Cloud Console.

- **License:** bring your own license (BYOL). Google does not bill for Cortx; get a license from
  XNODE at <https://www.xnode.ai/get-license> (licensing@xnode.ai). Google Cloud bills the GKE
  infrastructure separately.
- **End User License Agreement:** <https://www.xnode.ai/legal-eula>

## What gets installed

The Marketplace deployer creates a small app in the namespace you choose: an `Application`
resource, a Secret with your settings, and an installer. The installer installs Cortx with Helm:

- **Platform:** PostgreSQL (CloudNativePG), Redis, RabbitMQ, MinIO, Neo4j, Qdrant, HashiCorp Vault,
  NATS, Temporal, Prometheus, Grafana, Loki, Tempo
- **Cortx apps:** the Cortx microservices (installed once your license Secret exists)
- **Ingress:** Traefik, with one external load balancer for all Cortx hostnames
- **Storage:** `standard-rwo` persistent disks for databases; a bundled NFS provisioner
  (StorageClass `cortx-nfs`) for shared application volumes

Cortx creates its own namespaces (`cortx-platform`, `apps`, `vault`, `postgres`, `redis`, `rabbitmq`,
`minio-tenant`, `neo4j-community`, `nats`, `monitoring`, `temporal`, `nexus`, `pgvectors`) and
cluster-scoped resources for its operators (CRDs, ClusterRoles, webhooks). The installer's service
account is granted the cluster permissions it needs for that; see `schema.yaml`. Install one Cortx
instance per cluster.

## Prerequisites

- A Google Cloud project with billing enabled, and these tools: `gcloud`, `kubectl`, `docker`.
- A **GKE Standard** cluster, Kubernetes **1.33 or later**, Linux AMD64 nodes:
  **8 × `n2-standard-8`** (8 vCPU / 32 GiB) or larger. Cortx requests about 45 vCPU and 110 GiB.
- Project quota in your region (new projects often have less): about **64 vCPUs**, **1.5 TB SSD
  persistent disk** (`SSD_TOTAL_GB`) and **10 in-use IP addresses**. Use `pd-standard` node boot
  disks to keep SSD quota for Cortx volumes.
- A domain you control, and a **wildcard TLS certificate** for `*.<domain>` (PEM certificate with
  chain and private key).
- An **OpenAI API key**.
- Your **Cortx license bundle** (`cortx-license-bundle.yaml`), from XNODE.

Example cluster:

```shell
export PROJECT=<your-project> ZONE=us-central1-a CLUSTER=cortx
gcloud container clusters create "$CLUSTER" --project "$PROJECT" --zone "$ZONE" \
  --release-channel regular --machine-type n2-standard-8 --num-nodes 8 \
  --disk-type pd-standard --disk-size 100
gcloud container clusters get-credentials "$CLUSTER" --project "$PROJECT" --zone "$ZONE"
```

## Deploy from the command line

1. Install the Application CRD (once per cluster):

   ```shell
   kubectl apply -f "https://raw.githubusercontent.com/GoogleCloudPlatform/marketplace-k8s-app-tools/master/crd/app-crd.yaml"
   ```

2. Create a namespace and your license Secret:

   ```shell
   export NAMESPACE=cortx
   kubectl create namespace "$NAMESPACE"
   kubectl create secret generic cortx-license-bundle -n "$NAMESPACE" \
     --from-file=bundle=cortx-license-bundle.yaml
   ```

   Without this Secret, Cortx installs the platform only; the Cortx apps are added automatically
   as soon as the Secret is created.

3. Install the Google Cloud Marketplace tools (`mpdev`):

   ```shell
   mkdir -p ~/bin
   docker run --rm --entrypoint cat gcr.io/cloud-marketplace-tools/k8s/dev /scripts/dev > ~/bin/mpdev
   chmod +x ~/bin/mpdev
   ```

   If you use Docker Engine 29 or later, set `export EXTRA_DOCKER_PARAMS="-e DOCKER_API_VERSION=1.44"`
   before running `mpdev`.

4. Set your parameters. Use the deployer image and release shown on the Marketplace listing
   (**Command line deployment** tab):

   ```shell
   export DEPLOYER_IMAGE=<deployer image from the listing>:1.1
   export DOMAIN=cortx.example.com
   export TLS_CERT=$(base64 -w0 fullchain.pem)
   export TLS_KEY=$(base64 -w0 privkey.pem)
   export OPENAI_KEY=<your OpenAI API key>
   ```

5. Deploy:

   ```shell
   ~/bin/mpdev install --deployer="$DEPLOYER_IMAGE" --parameters="$(cat <<EOF
   {
     "name": "cortx",
     "namespace": "$NAMESPACE",
     "domain": "$DOMAIN",
     "tls.cert": "$TLS_CERT",
     "tls.key": "$TLS_KEY",
     "llm.openaiKey": "$OPENAI_KEY",
     "licenseSecretName": "cortx-license-bundle",
     "admin.email": "admin@example.com",
     "admin.firstName": "Admin",
     "admin.lastName": "User",
     "admin.mobile": "0000000000",
     "admin.entityName": "Example Inc",
     "admin.accountName": "example",
     "admin.country": "US"
   }
   EOF
   )"
   ```

   All deploy parameters, including optional Microsoft Entra ID, LDAP, SMTP, Slack, Anthropic and
   Whisper settings, are listed in [`schema.yaml`](schema.yaml). Internal passwords and API keys are
   generated automatically.

6. Follow the install (15–20 minutes):

   ```shell
   kubectl logs -f deployment/cortx-installer -n "$NAMESPACE"
   ```

   When it finishes, the log prints the ingress IP address.

7. Create a wildcard DNS record `*.<domain>` pointing to that IP, then open
   `https://prod.<domain>` (`<envName>.<domain>`, or the `webHost` you set) and sign in with the
   admin email you set.

## Upgrade

Each Cortx release is published as a new release on the Marketplace listing. Contact
itsupport@xnode.ai for the upgrade steps for your installed version before upgrading.

## Uninstall

```shell
kubectl delete application cortx -n "$NAMESPACE"
```

The installer removes Cortx (applications, operators, volumes and namespaces) before it exits; this
takes several minutes. Cortx CustomResourceDefinitions are kept, because other software on the
cluster may use them. Then delete the namespace and, if you created it for Cortx, the cluster.

## Repository contents

| Path | Purpose |
|---|---|
| `schema.yaml` | Deploy form and parameters (Google Cloud Marketplace schema v2) |
| `chart/cortx-deployer/` | Deployer chart: Application, settings Secret, installer |
| `LICENSE` | License notice |

## Support

Email itsupport@xnode.ai, or see <https://www.xnode.ai/contact>.
