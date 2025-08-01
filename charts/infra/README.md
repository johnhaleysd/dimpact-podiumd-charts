# Infra

## Rationale infra<>podiumd

Podiumd helm charts rely on pre-existence and availability of operators, storageclasses and components such as, but not limited to: NFS, Traefik proxy, Cloudnative-PG custom-resource-definitions, etc.
Because these MUST have lifecycles independent from Podiumd and its apps and services, they live in a separate, independent Helm installation called 'infra'.

You must install the _infra_ chart before installing _podiumd_.

## Setup

Some cluster resources are installed first.
Second, the database schemas are prepared before the _podiumd_ chart may be deployed.

### Cluster resources

From the root of this repo, run:

```bash
# (perhaps first:)
# helm repo add cnpg https://cloudnative-pg.github.io/charts
# helm upgrade --install cnpg --namespace cnpg-system --create-namespace cnpg/cloudnative-pg
#
helm dependency build infra charts/infra/
helm template infra charts/infra/ > infra-tpl.yaml

# First get the cnpg CRDs installed before the Infra chart can be installed:
yq '. | select(.kind=="CustomResourceDefinition")' < infra-tpl.yaml > infra-crds.yaml
kubectl create -f infra-crds.yaml

# Hand over management to Helm by setting labels and annotations:
for n in backups clusterimagecatalogs clusters databases imagecatalogs \
  poolers publications scheduledbackups subscriptions; do
     crd="crd/${n}.postgresql.cnpg.io"
     kubectl label --overwrite $crd app.kubernetes.io/managed-by=Helm
     kubectl annotate --overwrite $crd \
       meta.helm.sh/release-name=infra meta.helm.sh/release-namespace=default
done
```

All should install nicely. Once complete, get the EXTERNAL-IP assigned to the `infra-traefik` LoadBalancer service and use it to prepare and publish (wildcard) DNS records for the needed podiumd services, pointing to this IP.
Note: DNS records should exist before installing PodiumD services, or keycloak authentications will not work.
E.g.:

```bash
$ kubectl get svc/infra-traefik
NAME            TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
infra-traefik   LoadBalancer   10.244.73.2   45.12.50.8    80:30232/TCP,443:32417/TCP   7d19h
```

Finally, install **solr-cloud** CustomResourceDefinitions, as ZAC will use them to  install the zac-solr operator:

```bash
kubectl create -f https://solr.apache.org/operator/downloads/crds/v0.9.1/all-with-dependencies.yaml
```

## PodiumD install

Note: this may not be the ideal place to document these, however I could not find any existing instructions, so here goes.

### Namespace

```bash
# create payload namespace
kubectl create namespace podiumd
# find postgres cluster name and superuser password to prep and apply create-schemas.yaml:
kubectl create -n podiumd -f .../create-schemas.yaml
```

### Database schemas

The payload namespace and database schema are created next. These shall remain across podiumd releases.
You must view the cloudnative-pg superuser ConfigMap to find the postgres cluster name and superuser password generated during the _infra_ chart install.
Substitute these in .../create-schemas.yaml to allow the job to prepare schemas for all the apps and services that we are about to deploy:

```bash
# find postgres cluster name and superuser password to prep and apply create-schemas.yaml:
kubectl create -n podiumd -f .../create-schemas.yaml
```

### IngressRoutes

Traefik needs ingressroutes setup in advance, so that its tls-provisioner can fetch LetsEncrypt certificates.

```bash
kubectl -n podiumd apply -f .../ingress-kcX.yaml
```

### Infinispan Secrets

Before infinispan can work, we must publish unique Secrets, as mentioned in podiumd/values.yaml around line 152 re `# how to generate your own secret for use:`:

```bash
openssl req -nodes -x509 -sha256 -newkey rsa:4096 -keyout ispn.key -out ispn.crt -days 3560 -subj "/C=NL/ST=Noord Holland/L=Amsterdam/O=INFO/OU=integratieteam/CN=ispn" -addext "subjectAltName = DNS:ispn,DNS:ispn.podiumd,DNS:ispn.podiumd.svc.cluster.local"
openssl pkcs12 -export -out ispn.pfx -inkey ispn.key -in ispn.crt -name server
# use `password` as passphrase (as added under credentials add command in values.yaml infinispan.deploy.security.batch)
keytool -list -keystore ispn.pfx -storepass password
kubectl create secret generic ispn-secret --from-file=tls.crt=ispn.crt --from-file=tls.key=ispn.key --from-file=keystore.p12=ispn.pfx --from-literal=password=password --from-literal=alias=server -n podiumd
kubectl create secret generic ispn-transport-secret --from-file=tls.crt=ispn.crt --from-file=tls.key=ispn.key --from-file=cert.p12=ispn.pfx --from-literal=password=password --from-literal=alias=server -n podiumd
```

### Podiumd

Next up is installation of the _podiumd_ chart. As a good habit, make sure chart dependencies are up-to-date and ask helm to only generate templates (for you to verify), followed by a server-side dry-run before performing the actual upgrade/install. This may save you from having to undo partial upgrades and manually remove pvcs, pods or jobs.

This assumes you have a .../values-kcX.yaml file prepared with DNS names published and ready, pointing to the EXTERNAL-IP assigned to the `infra-traefik` services that was just deployed as part of the Infra chart.

```bash
helm dependency update charts/podiumd/

# (optional) only template out manifests for review:
helm template MY_RELEASE charts/podiumd/ -n podiumd --values .../values-kcX.yaml > podiumd-templates.yaml
# you may now use `yq -C . < podiumd-templates.yaml` or your favorite yaml viewer to verify the release.
# (optional) if satisfied, run a server-side dry-run
helm upgrade --install MY_RELEASE charts/podiumd/ -n podiumd --values .../values-kcX.yaml --dry-run=server
```

Next we start installing podiumd services for real.
If you feel lucky, you may enable every service at once, however it seems prudent to enable services one by one.

#### 1. Infinispan

We start with _infinispan_ - you'll want to make sure this starts and that it can access its secrets before continuing:

```bash
# only set "infinispan.enabled: true" in .../values-kcX.yaml and run:
helm upgrade --install MY_RELEASE charts/podiumd/ -n podiumd --values .../values-kcX.yaml
```

#### 2. Keycloak

Next, enable _keycloak_ by adding `keycloak.enabled: true` in .../values-kcX.yaml and make sure it runs properly:

```bash
# (optional) run a server-side dry-run
# helm upgrade --install MY_RELEASE charts/podiumd/ -n podiumd --values .../values-kcX.yaml --dry-run=server
helm upgrade --install MY_RELEASE charts/podiumd/ -n podiumd --values .../values-kcX.yaml
```

After a couple of minutes, Keycloak springs to life. Upon trying to login to <https://keycloak-admin.kc2.info.nl/>, you'll find that Azure Entra-federated login requires you to add these KC redirect-URLs to the OIDC config in the Azure Service Principal:

- <https://keycloak-admin.kc2.info.nl/realms/master/broker/oidc-admin-info/endpoint>
- <https://keycloak.kc2.info.nl/realms/podiumd/broker/oidc-info/endpoint>

Keycloak-Admin portal now accepts logins by the INFO "Keycloak podiumd admins" group! :D

#### 3. Objecttypen, Objecten, Openzaak, Opennotificaties

We continue with other services, such as _objecttypen_, _objecten_ and _openzaak_.

##### 3.1 Objecttypen

Set `objecttypen.enabled: true` in .../values-kcX.yaml, repeat the familiar `helm upgrade --install ...` command and make sure _objecttypen_ runs properly.

```bash
# you get the drill - the `helm upgrade ...`  commands don't change.
# after a minute, you should verify:
 $ kubectl get pod -n podiumd | grep objecttypen
create-required-objecttypen-job-h5p78   0/1     Completed   3          2m19s
objecttypen-7c4db747bc-d7bvh            1/1     Running     0          2m19s
objecttypen-redis-master-0              1/1     Running     0          2m19s
```

Logon to <https://objecttypen.kc2.info.nl/>, register your 2FA app and complete the first logon. You'll see the "Objecttypes Administration" welcome screen. Navigate to Data --> Object types and you should see 7 object types prepopulated. If you do, the _create-required-catalogi_ and _create-required-objecttypes_ jobs have completed.

##### 3.2 Objecten

Set `objecten.enabled: true` in .../values-kcX.yaml, repeat `helm upgrade --install ...` command(s) and make sure _objecten_ runs properly.
First, a _objecten-config_ job will run for a while. Upon finishing, the new set of pods should reach "Running" state:

```bash
# you get the drill - the `helm upgrade ...`  commands don't change.
# after a minute, you should check to see if the job that populates objecten runs and completes.:
$ kubectl get jobs -n podiumd
NAME                              STATUS     COMPLETIONS   DURATION   AGE
create-required-objecttypen-job   Complete   1/1           4s         2m44s
objecten-config                   Running    0/1           2m44s      2m44s
...
$ kubectl get jobs -n podiumd
NAME                              STATUS     COMPLETIONS   DURATION   AGE
create-required-objecttypen-job   Complete   1/1           4s         2m44s
$ # note the absence of the objecten-config job.
$ kubectl get pod -n podiumd|grep objecten
objecten-7fb7c497d6-8nv25               1/1     Running     0          6m43s
objecten-redis-master-0                 1/1     Running     0          6m43s
objecten-worker-f7776f6ff-tlw9g         1/1     Running     0          6m43s
```

Logon to <https://objecten.kc2.info.nl/>. You'll see the "Objects Administration" screen. Navigate to Data --> Objecten and you should see 0 objects and the optino to add your first.

##### 3.3 Openzaak

Set `openzaak.enabled: true` in .../values-kcX.yaml, repeat `helm upgrade --install ...` and make sure _openzaak_ runs properly.

```bash
# verify:
$ kubectl get pvc,deploy,svc -n podiumd|grep openzaak
persistentvolumeclaim/openzaak                                Bound    pvc-e1e1e883-5f68-4dc5-a52e-c1f523d84d84   1Gi        RWX            nfs             <unset>                 2m41s
persistentvolumeclaim/redis-data-openzaak-redis-master-0      Bound    pvc-db8e1c58-2a1c-4a93-85db-d5f6c73772be   8Gi        RWO            block-storage   <unset>                 2m39s
deployment.apps/openzaak            1/1     1            1           2m40s
deployment.apps/openzaak-beat       1/1     1            1           2m40s
deployment.apps/openzaak-nginx      1/1     1            1           2m40s
deployment.apps/openzaak-worker     1/1     1            1           2m40s
service/openzaak                     ClusterIP   10.244.107.162   <none>        80/TCP              2m40s
service/openzaak-nginx               ClusterIP   10.244.103.137   <none>        80/TCP              2m40s
service/openzaak-redis-headless      ClusterIP   None             <none>        6379/TCP            2m40s
service/openzaak-redis-master        ClusterIP   10.244.113.72    <none>        6379/TCP            2m40s
```

Logon to <https://openzaak.kc2.info.nl/>. You'll see the "Open Zaak startpunt" welcome screen. Click "Bekijk configuratie" and notice that NRC URL <https://notificaties.kc2.info.nl> is still down. We'll get to it later.
Following the red "Beheer" button from the welcome screen leads you to "Open Zaak beheer", which should allow you to manage Open Zaak settings, groups, etc.

##### 3.4 Opennotificaties

Set `opennotificaties.enabled: true` in .../values-kcX.yaml, repeat `helm upgrade --install ...` and verify _openzaak_:

```bash
# a lot of extra resources should quickly appear and reach "Bound" (pvc) or "Running" (pods) state:
$ kubectl get all,pvc -n podiumd|grep opennotificaties
pod/opennotificaties-beat-69889fff6c-rgfwz     1/1     Running     0             2m13s
pod/opennotificaties-dfc754755-hb96s           1/1     Running     0             2m13s
pod/opennotificaties-rabbitmq-0                1/1     Running     0             2m12s
pod/opennotificaties-redis-master-0            1/1     Running     0             2m12s
pod/opennotificaties-worker-7f4d4b6b8f-xs6sc   1/1     Running     0             2m13s
service/opennotificaties                     ClusterIP   10.244.98.65     <none>        80/TCP                                  2m13s
service/opennotificaties-rabbitmq            ClusterIP   10.244.125.102   <none>        5672/TCP,4369/TCP,25672/TCP,15672/TCP   2m13s
service/opennotificaties-rabbitmq-headless   ClusterIP   None             <none>        4369/TCP,5672/TCP,25672/TCP,15672/TCP   2m13s
service/opennotificaties-redis-headless      ClusterIP   None             <none>        6379/TCP                                2m13s
service/opennotificaties-redis-master        ClusterIP   10.244.108.196   <none>        6379/TCP                                2m13s
deployment.apps/opennotificaties          1/1     1            1           2m13s
deployment.apps/opennotificaties-beat     1/1     1            1           2m13s
deployment.apps/opennotificaties-worker   1/1     1            1           2m13s
replicaset.apps/opennotificaties-beat-69889fff6c     1         1         1       2m13s
replicaset.apps/opennotificaties-dfc754755           1         1         1       2m13s
replicaset.apps/opennotificaties-worker-7f4d4b6b8f   1         1         1       2m13s
statefulset.apps/opennotificaties-rabbitmq       1/1     2m12s
statefulset.apps/opennotificaties-redis-master   1/1     2m12s
persistentvolumeclaim/opennotificaties                             Bound    pvc-2209e6a1-fde1-42d5-80bb-3976f5ae5fcf   1Gi        RWX            nfs             <unset>                 2m14s
persistentvolumeclaim/redis-data-opennotificaties-redis-master-0   Bound    pvc-cdef4aa2-cf34-4570-8e93-54706b71ebe3   1Gi        RWO            block-storage   <unset>                 2m12s
```

From "Open Zaak startpunt", return to "Bekijk configuratie" and notice that NRC connection now reports "Kan kanalen ophalen". ;)
<https://notificaties.kc2.info.nl/> offers "Bekijk configuratie" and "Beheer". Check config first - it should report that it "Can retrieve authotizations" and "Can retrieve kanalen". It does not yet "Listen to AC notifications" because wen have not deployed openzaak yet.
Back on the welcome screen follow "Beheer" and navigate the menu to "Notificaties" --> "Abonnementen". Two subscribers should be visible.

#### 4 Remaining services

These remaining services should now be deployed:

- openformulieren
- openklant
- apiproxy
- zac

##### 4.1 Openformulieren

Set `openformulieren.enabled: true` in .../values-kcX.yaml, repeat `helm upgrade --install ...` and verify:

```bash
# Many new pods will appear. First nginx will start, then redis, after which a job.batch/openformulieren-config should run. This takes a number of minutes to complete.
$ k get all,pvc -n podiumd|grep formulier
pod/openformulieren-694d5b7c94-p8tfw           1/1     Running     0             4m30s
pod/openformulieren-beat-5c5597b7d6-9fwvb      1/1     Running     0             4m30s
pod/openformulieren-nginx-ffbcb5879-wmsgs      1/1     Running     0             25m
pod/openformulieren-redis-master-0             1/1     Running     0             25m
pod/openformulieren-worker-587d9859bb-hbd6s    1/1     Running     0             4m30s
service/openformulieren                      ClusterIP   10.244.113.99    <none>        80/TCP                                  25m
service/openformulieren-nginx                ClusterIP   10.244.112.78    <none>        80/TCP                                  25m
service/openformulieren-redis-headless       ClusterIP   None             <none>        6379/TCP                                25m
service/openformulieren-redis-master         ClusterIP   10.244.118.4     <none>        6379/TCP                                25m
deployment.apps/openformulieren           1/1     1            1           25m
deployment.apps/openformulieren-beat      1/1     1            1           25m
deployment.apps/openformulieren-nginx     1/1     1            1           25m
deployment.apps/openformulieren-worker    1/1     1            1           25m
replicaset.apps/openformulieren-694d5b7c94           1         1         1       4m31s
replicaset.apps/openformulieren-778cfb4              0         0         0       25m
replicaset.apps/openformulieren-beat-57474d4df8      0         0         0       25m
replicaset.apps/openformulieren-beat-5c5597b7d6      1         1         1       4m31s
replicaset.apps/openformulieren-nginx-ffbcb5879      1         1         1       25m
replicaset.apps/openformulieren-worker-57f7bf489d    0         0         0       25m
replicaset.apps/openformulieren-worker-587d9859bb    1         1         1       4m31s
statefulset.apps/openformulieren-redis-master    1/1     25m
persistentvolumeclaim/openformulieren                              Bound    pvc-4699cc14-a1a9-44bd-acd4-c07627e22e9f   1Gi        RWX            nfs             <unset>                 25m
persistentvolumeclaim/redis-data-openformulieren-redis-master-0    Bound    pvc-eb49cf5d-80de-4210-91b7-030c0773eb0f   1Gi        RWO            block-storage   <unset>                 25m
```

Because <https://formulieren.kc2.info.nl/> shows a friendly HTTP 403, we verify <https://formulier.kc2.info.nl/admin> instead. It should accept Azure-federated logins. 

##### 4.2 Openklant

Set `openklant.enabled: true` in .../values-kcX.yaml, repeat `helm upgrade --install ...` and verify:

```bash
$ kubectl get all,pvc -n podiumd|grep klant
pod/openklant-798d8ff65-6tv9z                  1/1     Running     0             2m22s
pod/openklant-nginx-6f8b556f65-tgcph           1/1     Running     0             2m22s
pod/openklant-redis-master-0                   1/1     Running     0             2m21s
pod/openklant-worker-6ff9997b4b-fjj8t          1/1     Running     0             2m22s
service/openklant                            ClusterIP   10.244.76.245    <none>        80/TCP                                  2m23s
service/openklant-nginx                      ClusterIP   10.244.116.147   <none>        80/TCP                                  2m23s
service/openklant-redis-headless             ClusterIP   None             <none>        6379/TCP                                2m23s
service/openklant-redis-master               ClusterIP   10.244.74.185    <none>        6379/TCP                                2m23s
deployment.apps/openklant                 1/1     1            1           2m22s
deployment.apps/openklant-nginx           1/1     1            1           2m22s
deployment.apps/openklant-worker          1/1     1            1           2m22s
replicaset.apps/openklant-798d8ff65                  1         1         1       2m22s
replicaset.apps/openklant-nginx-6f8b556f65           1         1         1       2m22s
replicaset.apps/openklant-worker-6ff9997b4b          1         1         1       2m22s
statefulset.apps/openklant-redis-master          1/1     2m21s
persistentvolumeclaim/openklant                                    Bound    pvc-7bb6d203-49e3-49d1-aa49-007ec2ec6340   1Gi        RWX            nfs             <unset>                 2m24s
persistentvolumeclaim/redis-data-openklant-redis-master-0          Bound    pvc-57b2b33a-31ca-4027-af83-5966a03cd8c9   1Gi        RWO            block-storage   <unset>                 2m21s
```

Afterwards, check <https://openklant.kc2.info.nl/> and logon to the Beheer dashboard.

##### 4.3 Apiproxy

ZAC will need _api-proxy_ for bagApi, brpApi and kvkApi.

Set `apiproxy.enabled: true` in .../values-kcX.yaml, repeat `helm upgrade --install ...` and verify services are alive (there is no frontend URL to check).

```bash
$ kubectl get deploy api-proxy -n podiumd -o wide
NAME        READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES         SELECTOR
api-proxy   1/1     1            1           4m41s   nginx        nginx:1.27.4   app=api-proxy
```

##### 4.4 ZAC

At long last we deploy ZAC - specifically: _zac_, _zac-office-converter_, _zac-opa_ (Open Policy Agent), Solr-core- and Zookeeper operators and cronjobs.

**FIXME-FIXME-FIXME**

Until the **zac-solr** storageClass can be controlled to use a specific `storageClass: null` for the pvcTemplates of statefulset.apps _zac-solr-solrcloud_ and  _zac-solr-solrcloud-zookeeper_, we can setup an **alias-storageClass**, e.g.:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi
  labels:
    purpose: alias
provisioner: bs.csi.gridscale.io
parameters:
  type: storage
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Install it using:

```bash
kubectl create -f .../alias-storageclass.yaml
```

Now we can deploy ZAC and accompanying resources.

Set `zac.enabled: true` in .../values-kcX.yaml, repeat `helm upgrade --install ...` and verify the results, e.g.:

```bash
# make sure that PVCs get "Bound":
$ kubectl get pvc -n podiumd | egrep solr\|zookeeper
data-zac-solr-solrcloud-0                    Bound    pvc-b96c2547-fdee-4274-83ab-97cbb075920b   1Gi        RWO            managed-csi     <unset>                 41m
data-zac-solr-solrcloud-1                    Bound    pvc-8b526ce9-5d62-44dd-99ca-177e7360d9f4   1Gi        RWO            managed-csi     <unset>                 41m
data-zac-solr-solrcloud-2                    Bound    pvc-7d1ab905-620f-46ae-a2bf-08b1f792bb03   1Gi        RWO            managed-csi     <unset>                 41m
data-zac-solr-solrcloud-zookeeper-0          Bound    pvc-adf58a0e-dbbe-4c81-a64c-276e49683f8c   1Gi        RWO            managed-csi     <unset>                 41m
#
# ...and see that statefulsets and deployments reach desired numbers of replicas:
$ kubectl get all -n podiumd | egrep solr\|zookeeper\|zac
pod/solr-operator-5fbb946b44-q8llx             1/1     Running            0               60m
pod/zac-755957d5f-qc4v9                        0/1     Running            8 (3m11s ago)   60m
pod/zac-nginx-576c7f945c-kjwqk                 0/1     CrashLoopBackOff   16 (3m1s ago)   60m
pod/zac-office-converter-869dc68f65-wknmp      1/1     Running            0               60m
pod/zac-opa-574465cb85-rfxt8                   1/1     Running            0               60m
pod/zac-solr-solrcloud-0                       1/1     Running            0               60m
pod/zac-solr-solrcloud-1                       1/1     Running            1 (35m ago)     60m
pod/zac-solr-solrcloud-2                       1/1     Running            1 (35m ago)     60m
pod/zac-solr-solrcloud-zookeeper-0             1/1     Running            0               60m
pod/zookeeper-operator-79586bb5bc-thwkm        1/1     Running            0               60m
service/zac                                         ClusterIP   10.244.87.203    <none>        80/TCP                                         60m
service/zac-nginx                                   ClusterIP   10.244.105.171   <none>        80/TCP                                         60m
service/zac-office-converter                        ClusterIP   10.244.120.39    <none>        80/TCP                                         60m
service/zac-opa                                     ClusterIP   10.244.109.45    <none>        8181/TCP                                       60m
service/zac-solr-solrcloud-common                   ClusterIP   10.244.101.14    <none>        80/TCP                                         60m
service/zac-solr-solrcloud-headless                 ClusterIP   None             <none>        8983/TCP                                       60m
service/zac-solr-solrcloud-zookeeper-admin-server   ClusterIP   10.244.67.75     <none>        8080/TCP                                       60m
service/zac-solr-solrcloud-zookeeper-client         ClusterIP   10.244.106.209   <none>        2181/TCP                                       60m
service/zac-solr-solrcloud-zookeeper-headless       ClusterIP   None             <none>        2181/TCP,2888/TCP,3888/TCP,7000/TCP,8080/TCP   60m
deployment.apps/solr-operator             1/1     1            1           60m
deployment.apps/zac                       0/1     1            0           60m
deployment.apps/zac-nginx                 0/1     1            0           60m
deployment.apps/zac-office-converter      1/1     1            1           60m
deployment.apps/zac-opa                   1/1     1            1           60m
deployment.apps/zookeeper-operator        1/1     1            1           60m
replicaset.apps/solr-operator-5fbb946b44             1         1         1       60m
replicaset.apps/zac-755957d5f                        1         1         0       60m
replicaset.apps/zac-nginx-576c7f945c                 1         1         0       60m
replicaset.apps/zac-office-converter-869dc68f65      1         1         1       60m
replicaset.apps/zac-opa-574465cb85                   1         1         1       60m
replicaset.apps/zookeeper-operator-79586bb5bc        1         1         1       60m
statefulset.apps/zac-solr-solrcloud              3/3     60m
statefulset.apps/zac-solr-solrcloud-zookeeper    1/1     60m
cronjob.batch/zac-sig-del      30 8 * * 1-5   <none>     False     0        <none>          60m
cronjob.batch/zac-signaleren   0 8 * * 1-5    <none>     False     0        <none>          60m
```

**FIXME-FIXME: Note the _zac-nginx_ pod in CrashLoopBackoff**

