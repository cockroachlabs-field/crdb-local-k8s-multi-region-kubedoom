# CockroachDB Multi-Cloud Demo (Local k3d)

## Install k3d on your Mac
To run a multi regional CockroachDB locally on a Macbook you need something that will enable you to run Kubernetes locally. In this demo I will use k3d. To install k3d run the following command.

```
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

Create a cluster named `kube-doom` with just a single server node:
```
k3d cluster create kube-doom
```

Use the new cluster with kubectl, e.g.:
```
kubectl get nodes
```

## Deploy CockroachDB into the the k3d cluster
Create three variables with the region names desired.
```
export eks_region="eu-west-1"
export gke_region="europe-west4"
export aks_region="uksouth"
```

Create three separate namespaces.
```
kubectl create namespace $eks_region
kubectl create namespace $gke_region 
kubectl create namespace $aks_region
```

We are going to create the certificates required to deploy CockroachDB.
```
mkdir certs my-safe-directory
```

Create the certificate authority.
```
cockroach cert create-ca \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

Create the client certificate.
```
cockroach cert create-client \
root \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

Add these certificates as kubernetes secrets into each of the namespaces.
```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--namespace $eks_region
```

```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--namespace $gke_region
```

```
kubectl create secret \
generic cockroachdb.client.root \
--from-file=certs \
--namespace $aks_region
```

Create the node certificates for each region.
```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$eks_region \
cockroachdb-public.$eks_region.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$eks_region" \
"*.cockroachdb.$eks_region.svc.cluster.local" \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```
Add it as a secret in to the first namespace.
```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--namespace $eks_region
```

```
rm certs/node.crt
rm certs/node.key
```

Now do the same again for the second region.
```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$gke_region \
cockroachdb-public.$gke_region.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$gke_region" \
"*.cockroachdb.$gke_region.svc.cluster.local" \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

Create the secret in the second region.
```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--namespace $gke_region
```

```
rm certs/node.crt
rm certs/node.key
```

Now the finally the third region.
```
cockroach cert create-node \
localhost 127.0.0.1 \
cockroachdb-public \
cockroachdb-public.$aks_region \
cockroachdb-public.$aks_region.svc.cluster.local \
"*.cockroachdb" \
"*.cockroachdb.$aks_region" \
"*.cockroachdb.$aks_region.svc.cluster.local" \
--certs-dir=certs \
--ca-key=my-safe-directory/ca.key
```

Upload the secret.
```
kubectl create secret \
generic cockroachdb.node \
--from-file=certs \
--namespace $aks_region
```

```
rm certs/node.crt
rm certs/node.key
```

Deploy the three separate StatefulSet.
> There are some hard codes region names in these files. If you have changed the region names you will need to edit these files.
```
kubectl -n $eks_region apply -f manifest/aws-cockroachdb-statefulset-secure.yaml
kubectl -n $gke_region apply -f manifest/gke-cockroachdb-statefulset-secure.yaml
kubectl -n $aks_region apply -f manifest/azure-cockroachdb-statefulset-secure.yaml
```

Once the pods are deployed we need to initialize the cluster. This is done by 'execing' into the container and running the `cockroach init` command.
```
kubectl exec \
--namespace $eks_region \
-it cockroachdb-0 \
-- /cockroach/cockroach init \
--certs-dir=/cockroach/cockroach-certs
```

Check that all the pods have started successfully.
```
kubectl get pods --namespace $eks_region
kubectl get pods --namespace $gke_region
kubectl get pods --namespace $aks_region
```

Next, create a secure client in the first region.
```
kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/client-secure.yaml --namespace $eks_region
```

Now exec into the pod using the Cockraoch SQL client.
```
kubectl exec -it cockroachdb-client-secure -n $eks_region -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```
Create a user and make that user an admin.
```
CREATE USER craig WITH PASSWORD 'cockroach';
GRANT admin TO craig;
\q
```

To user the map you need an enterprise license. Use the command below to cut a license key.
```
./crl-lic -type Evaluation -org "Cockroach Labs" -site -expiration "2023-10-03 00:00 America/New_York"
```

Exec into the pod again and apply the license.
```
kubectl exec -it cockroachdb-client-secure -n $eks_region -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```
Here are a couple of example cluster settings.
```
SET CLUSTER SETTING cluster.organization = 'Cockroach Labs';
SET CLUSTER SETTING enterprise.license = 'crl-0-EMC86ZkGGAIiDkNvY2tyb2FjaCBMYWJz';
```

To make use of the map we need to add the system locations below. You need to be exec'ed into the pod still.
```
INSERT INTO system.locations VALUES
  ('region', 'uksouth', 50.941, -0.799),
  ('region', 'europe-west4', 53.4386, 6.8355),
  ('region', 'eu-west-1', 53.142367, -7.692054);

```

Port forward port `8080` to localhost so you are able to access the CockroachDB Admin UI from the browser.
```
kubectl port-forward cockroachdb-0 8080 -n $eks_region
```

`Test this`
```
kubectl apply -f aws-svc-admin-ui.yaml --namespace $eks_region
kubectl apply -f gke-svc-admin-ui.yaml --namespace $gke_region
kubectl apply -f azure-svc-admin-ui.yaml --namespace $aks_region
```


## Kube-doom Setup

Deploy Kube-Doom using Kustomize.

```
kubectl apply -k manifest/
```

Open another terminal window and port forward port `5900` so you are able to use VNC Viewer to connect to the pod running kube-doom.
```
kubedoom_pod=$(kubectl get pods -n kubedoom -o name --no-headers=true)
echo $kubedoom_pod
kubectl port-forward -n kubedoom $kubedoom_pod 5900
```

VNC Password:
```
idbehold
```

Doom Cheats: 
```
IDDQD - invulnerability
IDKFA - full health, ammo, etc
IDSPISPOPD - no clipping / walk through walls
```

## Clean Up
To clean up the resources delete the cluster and delete the certificates.
```
k3d cluster delete
rm -R certs my-safe-directory
```
