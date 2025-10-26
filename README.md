# k8s-personal-boilerplate

# Table of Contents

- [k8s-personal-boilerplate](#k8s-personal-boilerplate)
- [Table of Contents](#table-of-contents)
- [Overview](#overview)
- [How to Setup](#how-to-setup)
- [Other Information](#other-information)
  - [How to View Information from Helm Chart Repository](#how-to-view-information-from-helm-chart-repository)
  - [How to clean up Kubernetes objects](#how-to-clean-up-kubernetes-objects)
  - [How to connect to a Kubernetes nodes](#how-to-connect-to-a-kubernetes-nodes)
  - [How to start or connect to a Kubernetes pods](#how-to-start-or-connect-to-a-kubernetes-pods)
  - [How to monitor pod status](#how-to-monitor-pod-status)
  - [How to check resource requests/limits across nodes and pods](#how-to-check-resource-requestslimits-across-nodes-and-pods)
  - [How to set resource requests/limits](#how-to-set-resource-requestslimits)
  - [How to make services accessible locally (outside of Kubernetes cluster) for development](#how-to-make-services-accessible-locally-outside-of-kubernetes-cluster-for-development)
  - [How to check permission of a service account/role](#how-to-check-permission-of-a-service-accountrole)
  - [How to shutdown node or node pool](#how-to-shutdown-node-or-node-pool)
  - [How to version lock Helm chart](#how-to-version-lock-helm-chart)
  - [How to check if a Prometheus Service Monitor is picked up](#how-to-check-if-a-prometheus-service-monitor-is-picked-up)
  - [How to check if logs from particular namespaces/pods/services are picked up](#how-to-check-if-logs-from-particular-namespacespodsservices-are-picked-up)
  - [How to Back Up Using Velero](#how-to-back-up-using-velero)
  - [How to Access Kopia Back Up Without Velero](#how-to-access-kopia-back-up-without-velero)
  - [How to Back Up \& Migrate Wordpress](#how-to-back-up--migrate-wordpress)
  - [How to migrate postgresql database (between two running databases)](#how-to-migrate-postgresql-database-between-two-running-databases)
  - [How to migrate postgresql database (turn off old and turn on new database)](#how-to-migrate-postgresql-database-turn-off-old-and-turn-on-new-database)
  - [Airflow fails when connecting to database](#airflow-fails-when-connecting-to-database)
  - [Airflow containers fails when custom requirements.txt is supplied](#airflow-containers-fails-when-custom-requirementstxt-is-supplied)
  - [How to test Postgresql connection](#how-to-test-postgresql-connection)
  - [How to get a list of active Postgresql connections](#how-to-get-a-list-of-active-postgresql-connections)
  - [How to test Redis connection](#how-to-test-redis-connection)
  - [How to enable additional inbound connections (external of Kubernetes)](#how-to-enable-additional-inbound-connections-external-of-kubernetes)
  - [How to insert/get secrets into HashiCorp Vault](#how-to-insertget-secrets-into-hashicorp-vault)

# Overview

This repo is used to manage all the underlying Kubernetes setup and app deployment related to my personal websites.

Each folder represents either Kubernetes setup or an app deployment using Helm Chart + Helmfile.

Note that Digital Ocean load balancers and volumes are created via `kubectl`.

# How to Setup

1. Install `kubectl`, `helm`, `helmfile` and `doctl`
   1. `kubectl` - https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
   2. `helm` - https://helm.sh/docs/intro/install/
   3. `doctl` - https://docs.digitalocean.com/reference/doctl/how-to/install/
2. Install `helm` plugins
   1. `helmfile` - https://github.com/helmfile/helmfile
      1. Run `helmfile init` to install its dependencies
3. Setup Digital Ocean personal access token for `doctl` authentication
   1. Create a Personal Access Token at https://cloud.digitalocean.com/account/api/tokens
   2. Get access to kubernetes, block_storage and load_balancer at least
   3. `doctl auth init --context k8s-personal` - to setup
   4. `nano ~/.config/doctl/config.yaml` - manually update the access token under auth_context
   5. `doctl kubernetes cluster list` - test if this runs without error
4. Create Kubernetes cluster via Digital Ocean webUI
   1. https://cloud.digitalocean.com/kubernetes/clusters
5. Connect to created Kubernetes cluster so that `kubectl` and `helm` connects to cluster directly
   1. Using automatic approach - `doctl kubernetes cluster kubeconfig save {digital-ocean-cluster-id}`
6. Create 3 node pools using Digital Ocean web UI or CLI.
   1. `master-k8s-node-pool` - 1 node ($48, 4 CPU, 8GB RAM)
   2. `worker-k8s-node-pool` - 2 nodes ($48, 4 CPU, 8GB RAM)
   3. `executor-small-k8s-node-pool` - 0 node, autoscaling ($84, 2 CPU, 16GB RAM)
7. Label all nodes from `master-k8s-node-pool` for core operations (e.g. ingress, monitoring).
   1. `doctl kubernetes cluster node-pool update k8s-personal-sfo3-cluster master-k8s-node-pool --label "node-type=master"`
   2. `kubectl get nodes --show-labels` to check labels. (There may be some delays for the labels being reflected correctly)
8. Label all nodes from `worker-k8s-node-pool` for other worker services.
   1. `doctl kubernetes cluster node-pool update k8s-personal-sfo3-cluster worker-k8s-node-pool --label "node-type=worker"`
   2. `kubectl get nodes --show-labels` to check labels. (There may be some delays for the labels being reflected correctly)
9. Label all nodes from `executor-small-k8s-node-pool` for on-demand execution tasks.
   1. `doctl kubernetes cluster node-pool update k8s-personal-sfo3-cluster executor-small-k8s-node-pool --label "node-type=executor-small"`
   2. `kubectl get nodes --show-labels` to check labels. (There may be some delays for the labels being reflected correctly)
   3. WARN if this is a fresh kubernetes setup, scale from zero may fail (due to Bitnami charts requesting ephemeral storage). In this case manually start one node then shutdown the node afterwards. Then autoscaling from zero should start working.
      1. `doctl kubernetes cluster node-pool update k8s-personal-sfo3-cluster executor-small-k8s-node-pool --count [1/0]`
   4. Note that Kubernetes cluster autoscaler will only scale down a node after 10 mins of low utilisation (50% of CPU & memory) and all pods are movable.
10. For each app/directory managed by a `helmfile.yaml`, version lock the dependencies if `helmfile.lock` is not present.
   1. `helmfile deps`
11. Install `ingress-nginx` to setup load balancer and ingress (DO-specific config present).
   1. `cd ingress-nginx`
   2. `helmfile sync`
   3. **NOTE** Uninstalling will results in new IP address given when restarted. Will need to update DNS.
12. Install `cert-manager` to manage TLS/SSL certificates.
   1. `cd cert-manager`
   2. `helmfile sync`
   3. During first run, `cert-manager` post install may fail, as the service takes some time to spin up.
13. Install `hashicorp-vault` to store secrets.
   1.  `cd hashicorp-vault`
   2.  `helmfile sync`
   3.  Manual init and unseal vault
       1.  `kubectl exec -it -n hashicorp-vault hashicorp-vault-0 -- sh`
       2.  `vault operator init` - note down all 5 unseal keys and initial root token
       3.  `export VAULT_TOKEN=<ROOT_TOKEN_VALUE>` - require for admin operations below
       4.  `vault operator unseal` - repeat 3 times
   4.  To setup vault for the first time
       1.  `vault secrets enable -path=secret/ -version=2 kv` - this uses kv engine v2
   5.  `vault status` - check that initialized is true and sealed is false
14. Insert secrets into `hashicorp-vault`.
   1. Create `dockerhub-secret` using POST API
      1. ```bash
            curl --header "X-Vault-Token: <ROOT_TOKEN_VALUE>" --request POST \
            --data '{"data": {"key":{"auths":{"https://index.docker.io/v1/":{"auth":"<BASE64-ENCODED_USERNAME:PASSWORD>","email": "<EMAIL>","password": "<PASSWORD>","username": "<USERNAME>"}}}}}' \
            https://vault.mydomain.com/v1/secret/data/dockerhub-secret
         ```
      2. To get base64-encode username and password, run `echo '<USERNAME>:<PASSWORD>' | base64`
15. Install `external-secrets` to import and manage secrets.
   1. `kubectl create secret generic vault-token -n external-secrets --from-literal=token=<YOUR_VAULT_TOKEN>` - this will be used by external-secrets to connect to vault.
   2. `cd external-secrets`
   3. `helmfile apply`
   4. `set -a; . .env.external-secrets; set +a; envsubst < ./resources/external-secrets-cluster-secret-store.yaml | kubectl apply -f -`
   5. Actual secrets should be created per application as required since they should be namespace specific.
      1. **NOTE** Uninstalling `external-secrets` will lead to deletion of all downstream `external-secrets` managed secrets.
16. Install `metrics-server` to enable node/pod resource monitoring.
   1. `cd metrics-server`
   2. `helmfile sync`
17. Install `prometheus` to enable node/pod monitoring.
   1. `cd prometheus`
   2. `helmfile sync`
      1. Can ignore the error `Secret in version "v1" cannot be handled as a Secret: illegal base64 data at input byte 0`.
18. Install `grafana-loki` to collect logs.
    1.  `cd grafana-loki`
    2.  `helmfile sync`
19. Install `grafana` to visualise node/pod metrics.
    1. `cd grafana`
    2. `set -a; . .env.grafana; set +a; helmfile sync`
20. Install `velero` to backup the cluster.
    1.  `cd velero`
    2.  `set -a; . .env.velero; set +a; helmfile sync`
21. Setup DNS records for domain names to point to the Kubernetes external IP (from load balancer).
   1.  Digital Ocean load balancer can only be associated with one domain name. So for any additional domain names, create a CNAME that points to the associated domain name (instead of directly to the IP).
22. Install `postgresql`.
   1.  `cd postgresql`
   2.  `kubectl create secret generic pgbouncer-env -n postgresql --from-env-file=.env.pgbouncer`
   3.  `kubectl create secret generic init-env -n postgresql --from-env-file=.env.postgresql`
   4.  `kubectl create secret generic init-script -n postgresql --from-file=./configs/init.sh`
   5.  `set -a; . .env.postgresql; set +a; helmfile sync` (NOTE can ignore the command not found error. NOTE some env variables have same names)
23. Install `redis`.
   1.  `cd redis`
   2.  `set -a; . .env.redis; set +a; helmfile sync`
24. Install `mariaDB`.
   1.  `cd mariadb`
   2.  `set -a; . .env.mariadb; set +a; helmfile sync`
25. Install `wordpress`.
   1. `cd wordpress`
   2. `set -a; . .env.wordpress; set +a; helmfile sync`
   3. [Note on manual plugin activation]
   4. [Note on where base files come from]
26. Install `homepage` app.
   1. `cd homepage`
   2. `kubectl create secret generic homepage-env -n homepage --from-env-file=.env.homepage`
   3. `helmfile sync`
27. Install `dash-dashboard` app.
   1. `cd dash-dashboard`
   2. `kubectl create secret generic dash-dashboard-env -n dash-dashboard --from-env-file=.env.dash-dashboard`
   3. `helmfile sync`
28. Install `airflow` app.
   1.  Note that upgrading this has an impact on `mydomain-airflow` and all associated repos due to Python and Airflow version dependencies.
   2.  `cd airflow`
   3.  # `kubectl create secret generic dag-general-env -n airflow --from-env-file=.env.dag.general`
   4.  # `kubectl create secret generic dag-hdb-resale-env -n airflow --from-env-file=.env.dag.hdb-resale`
   5.  `set -a; . .env.airflow; set +a; helmfile apply`
   6.  To start a local Airflow instance for testing, run `docker run apache/airflow:slim-3.0.3rc6-python3.12 standalone`
29. Install `mlflow` app.
   1.  Note that upgrading this has an impact on `mydomain-airflow` and all associated repos due to Python and MLFlow version dependencies.
   2.  `cd mlflow`
   3.  `set -a; . .env.mlflow; set +a; helmfile apply`

# Other Information

## How to View Information from Helm Chart Repository

1. Add new Helm chart repo
   1. `helm repo add jetstack https://charts.jetstack.io`
2. Search for specific charts in the repo
   1. `helm search repo jetstack` - list all charts in the repo if no search term is specified

## How to clean up Kubernetes objects

1. Delete all Kubernetes objects from a namespace (special objects like secrets will be kept)
   1. `kubectl delete all --all -n {namespace-name}`
   2. `kubectl delete namespace {namespace-name}` - this may work when the command above does not
2. Delete all Kubernetes objects from all namespaces
   1. `kubectl delete --all namespaces`
3. Delete all releases if using `helmfile`
   1. `helmfile destroy`
4. If there is a pod that get stuck in "Terminating" status for a long time, manuall delete them. Be careful when running this as the pod may not terminate properly after deletion.
   1. `kubectl delete pod --grace-period=0 --force --namespace {NAMESPACE} {POD_NAME}`

## How to connect to a Kubernetes nodes

1. Install `node-shell` (https://github.com/robscott/kube-capacity)
   1. Install `krew` - `https://krew.sigs.k8s.io/docs/user-guide/setup/install/`
   2. Install plugin with `krew` - `kubectl krew install node-shell`
2. Run `kubectl node-shell {NODE_NAME} --image docker.io/ubuntu:22.04`

## How to start or connect to a Kubernetes pods

1. To connect to existing containers in a running pod
   1. `kubectl exec -it -n {NAMESPACE_NAME} {POD_NAME} -- bash`
   2. `kubectl exec -it -n {NAMESPACE_NAME} {POD_NAME} -c {CONTAINER_NAME} -- bash` - connect to specific container
2. To start a new container in a running pod
   1. `kubectl debug -it -n {NAMESPACE_NAME} {POD_NAME} --image=docker.io/ubuntu:22.04 --target={CONTAINER_NAME}` - target container name refers to container to be debugged
3. To start a new pod
   1. `kubectl run -it -n {NAMESPACE_NAME} --rm --restart=Never ubuntu --image=docker.io/ubuntu:22.04 bash`

## How to monitor pod status

1. To get all pods that have restarted at least once
   1. `kubectl get pods --all-namespaces | awk '$5>0'`

## How to check resource requests/limits across nodes and pods

1. Install `kube-capacity` (https://github.com/robscott/kube-capacity)
   1. Needs to ensure that `metrics-server` is deployed and running.
   2. Install `krew` - `https://krew.sigs.k8s.io/docs/user-guide/setup/install/`
   3. Install plugin with `krew` - `kubectl krew install resource-capacity`
2. Run `kubectl resource-capacity --util` or `kubectl resource-capacity --util --pods`.
3. Get resource usage per container within a pod.
   1. `kubectl top pod -n {NAMESPACE} {POD_NAME} --containers`

## How to set resource requests/limits

1. Following advice from here - `https://home.robusta.dev/blog/stop-using-cpu-limits`
2. For CPU, always set requests but not limits.
3. For memory, always set requests and limits. And make requests equal limits.

## How to make services accessible locally (outside of Kubernetes cluster) for development

1. Setup port forwarding.
   1. `kubectl port-forward --namespace {NAMESPACE} svc/{SERVICE_NAME} {SOURCE_PORT}:{TARGET_PORT}`
   2. E.g. for prometheus, `kubectl port-forward --namespace prometheus svc/prometheus-prometheus 9090:9090`
2. Access web interface using browser at `http://127.0.0.1:{TARGET_PORT}/`

## How to check permission of a service account/role

1. Run `kubectl auth can-i --list --as=system:serviceaccount:{NAMESPACE}:{SERVICE_ACCOUNT} -n {TARGET_NAMESPACE}`
   1. E.g. `kubectl auth can-i --list --as=system:serviceaccount:prometheus:prometheus-prometheus -n wordpress`

## How to shutdown node or node pool

1. Setup new node or node pool as required (so that services can be automatically migrated over).
2. Prepare node for safe shutdown.
   1. `kubectl drain {NODE_NAME} --ignore-daemonsets --disable-eviction --delete-emptydir-data`
   2. This will delete all locally stored data (i.e. not on PV), and ignore PodDisruptionBudgets checks.
3. Check that the node is drained properly.
   1. `kubectl get nodes` (Drained nodes should be in `Ready,SchedulingDisabled` status.)
   2. `kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName={NODE_NAME}`
   3. Can ignore all kube-system pods and other node specific pods (e.g. prometheus node exporter).
4. Check that the pods have been migrated to new node properly.
   1. ```
      for n in $(kubectl get nodes -l {NODE_LABEL_KEY}={NODE_LABEL_VALUE} --no-headers | cut -d " " -f1); do
         kubectl get pods --all-namespaces  --no-headers --field-selector spec.nodeName=${n}
      done
   ```
5. Delete the node or node pool on DigitalOcean.
   1. If working with node pool, should drain all nodes under node pool one by one.

## How to version lock Helm chart

1. When under a directory managed by a `helmfile.yaml`
   1. Run `helmfile deps`
2. This will generate a lock file that will be used for subsequent helmfile installations.

## How to check if a Prometheus Service Monitor is picked up

1. Port forward prometheus and check on UI
   1. `kubectl port-forward --namespace prometheus svc/prometheus-prometheus 9090:9090`
2. Check using `kubectl` and `prometheus` secrets
   1. `kubectl get secrets -n prometheus prometheus-prometheus-prometheus -ojson | jq -r '.data["prometheus.yaml.gz"]' | base64 -d | gunzip | grep "{SERVICE_MONITOR_NAME}"`

## How to check if logs from particular namespaces/pods/services are picked up

1. Check that each node has a running promtail.
   1. `kubectl get pods -o wide -l app.kubernetes.io/component=promtail -n grafana-loki`
2. Port forward promtail and check on UI
   1. `kubectl port-forward --namespace grafana-loki {POD_NAME} 8080:8080`

## How to Back Up Using Velero

1. Make sure `velero` CLI is installed and setup properly.
   1. See https://velero.io/docs/v1.6/basic-install/#install-the-cli
   2. By default it will backup all Kubernetes resources & data in file systems (using kopia) onto Digital Ocean S3.
2. Create back up by running `velero backup create`
   1. [To backup specific namespace] `velero backup create wordpress-backup --include-namespaces wordpress`
   2. [To backup entire cluster] `velero backup create k8s-personal-cluster-backup`
   3. `velero backup describe wordpress-backup --details`
3. Restore from back up by running `velero restore create`
   1. `velero restore create --from-backup wordpress-backup`
   2. `velero restore describe wordpress-backup --details`
4. To schedule back up
   1. `velero schedule create k8s-personal-cluster-backup --schedule="0 2 * * Sun"`
   2. This will run every Sunday at 2am
5. To check schedule
   1. `velero schedule get`
6. To immediately trigger a scheduled job
   1. `velero backup create --from-schedule k8s-personal-cluster-backup`
7. To delete back up
   1. `velero backup delete wordpress-backup`

## How to Access Kopia Back Up Without Velero

1. Install `kopia` CLI.
   1. See https://kopia.io/docs/installation/#linux-installation-using-apt-debian-ubuntu
2. Connect to kopia repository on S3 bucket.
   1. `kopia repository connect s3 --bucket=k8s-personal-backup --endpoint="sfo3.digitaloceanspaces.com"  --access-key={ACCESS_KEY} --secret-access-key="{SECRET}" --prefix=kopia/mlflow/`
   2. The repository password is a static password common across all Velero+kopia instances, i.e. `static-passw0rd`
3. List the snapshots present in the repository & get the snapshot ID.
   1. A snapshot typically corresponds to one persistent volume or one local volume.
   2. `kopia snapshot list --all`
4. Restore/download the selected snapshot to local directory.
   1. `kopia snapshot restore {SNAPSHOT_ID} ./local_dir`

## How to Back Up & Migrate Wordpress

1. Backup WordPress site files (i.e. all files under WordPress root folder - /bitnami/wordpress)
   1. `kubectl exec -it -n wordpress {WORDPRESS_POD_NAME} -- bash`
   2. `cd /tmp && tar -cvzf wordpress-site-content-{DDMMYYYY}.tgz /bitnami/wordpress`
   3. `kubectl cp -n wordpress {WORDPRESS_POD_NAME}:/tmp/wordpress-site-content-{DDMMYYYY}.tgz ./backup/wordpress-site-content-{DDMMYYYY}.tgz`
2. Backup WordPress database
   1. `kubectl exec -it -n wordpress wordpress-mariadb-0 -- bash`
   2. `cd /tmp && mariadb-dump --add-drop-table -u root -p bitnami_wordpress > wordpress-database-{DDMMYYYY}.sql`
   3. `kubectl cp -n wordpress wordpress-mariadb-0:/tmp/wordpress-database-{DDMMYYYY}.sql ./backup/wordpress-database-{DDMMYYYY}.sql`
3. Import Wordpress database into new database
   1. `kubectl cp -n wordpress ./backup/wordpress-database-{DDMMYYYY}.sql wordpress-mariadb-0:/tmp/wordpress-database-{DDMMYYYY}.sql`
   2. `kubectl exec -it -n wordpress wordpress-mariadb-0 -- bash`
   3. `mariadb -u root -p bitnami_wordpress < /tmp/wordpress-database-{DDMMYYYY}.sql`
4. Extract WordPress site files
   1. `kubectl cp -n wordpress ./backup/wordpress-site-content-{DDMMYYYY}.tgz {WORDPRESS_POD_NAME}:/tmp/wordpress-site-content-{DDMMYYYY}.tgz`
   2. `kubectl exec -it -n wordpress {WORDPRESS_POD_NAME} -- bash`
   3. `cd /tmp/ && tar -xzf wordpress-site-content-{DDMMYYYY}.tgz`
   4. `rm var/www/html/wp-config.php` - remove backup config. Only applicable during migration between different setup.
   5. `cp -r ./* /bitnami/wordpress/`
5. Check that all Wordpress services are in ready state without errors
   1.  `kubectl get all -n wordpress`

## How to migrate postgresql database (between two running databases)

1. Make sure both source (i.e. old) and target (i.e. new) postgresql databases are able to accept remote connections.
2. Create corresponding database at target postgresql database.
   1. `CREATE DATABASE <db_name>`
3. Run `pg_dump` directly to dump data to remote. Should connect direct to postgresql if possible, not via pgbouncer for this operation.
   1. `PGPASSWORD=<source_password> pg_dump -h <source_host> -p <source_port> -U <source_user> <source_db> | PGPASSWORD=<target_password> psql -h <target_host> -p <target_port> -U <target_user> <target_db>`
   2. Then input password for `pg_dump` first, then for `psql`.
   3. Note that this operation will appear to hang for a bit before showing progress.

## How to migrate postgresql database (turn off old and turn on new database)

1. Backup data in old database using `pg_dumpall`.
   1. `kubectl exec -it -n postgresql postgresql-0 -- bash`
   2. `pg_dumpall -U postgres > /tmp/postgresql-database-{DDMMYYYY}.sql` (NOTE have to input password multiple times - once per database)
   3. `kubectl cp -n postgresql postgresql-0:/tmp/postgresql-database-{DDMMYYYY}.sql ./backup/postgresql-database-{DDMMYYYY}.sql`
2. Update `helmfile` and other yamls as needed and redeployed updated postgresql.
3. Restore data in new database using `psql`.
   1. `kubectl cp -n postgresql ./backup/postgresql-database-{DDMMYYYY}.sql postgresql-0:/tmp/postgresql-database-{DDMMYYYY}.sql`
   2. `kubectl exec -it -n postgresql postgresql-0 -- bash`
   3. `psql -U postgres -f /tmp/postgresql-database-{DDMMYYYY}.sql`

## Airflow fails when connecting to database

1. If Airflow deployment fails to start up while trying to connect to database, delete all tables in `airflow_db` using DBeaver.
2. This is more likely to happen when doing major upgrade or migration.

## Airflow containers fails when custom requirements.txt is supplied

1. If an Airflow deployment failed when a custom requirements.txt is supplied, this is because there are conflicts between the container package versions and the supplied package versions.
2. To resolve, go into a default Airflow container without custom requirements.txt, run `pip freeze` to get a copy of the container package version.
3. Then start an Airflow container with custom requirements.txt, wait for pip to ends and record the error messages.
   1. E.g. `apache-airflow-providers-google 15.1.0 requires pandas<2.2,>=2.1.2, but you have pandas 2.3.0 which is incompatible.`
4. Manually resolve these differences in the custom requirements.txt by adding or removing constraints, before rerunning it.

## How to test Postgresql connection

1. (Using postgresql CLI) Install postgresql CLI and run the command.
   1. `apt-get install postgresql-client`
   2. `pg_isready -d <db_name> -h <host_name> -p <port_number> -U <db_user>`

## How to get a list of active Postgresql connections

1. This is useful to debug cases when the database is overwhelmed with unclosed connections from clients.
2. To check a list of active Postgresql connections, run this
   1. `SELECT * FROM pg_stat_activity;`
3. To kill any active Postgresql connections, run this
   1. `SELECT pg_terminate_backend(<pg_stat_activity.pid>) FROM pg_stat_activity`

## How to test Redis connection

1. Install redis CLI.
   1. `apt-get install redis-tools`
2. (From external) Run the command, and if everything works, the response will be `PONG`.
   1. `redis-cli -h mydomain.com -p 6379 -a '[password]' PING`
3. (From within cluster) Start a debug/test container, then run the command, and if everything works, the response will be `PONG`.
   1. `redis-cli -h redis-master.redis.svc.cluster.local -p 6379 -a '[password]' PING`

## How to enable additional inbound connections (external of Kubernetes)

1. (If inbound connection is required for http or https traffic)
   1. Enable ingress via either helm chart values directly or create ingress object manually.
2. (If inbound connection is required for any other TCP/UDP traffic)
   1. Modify `ingress-nginx-values.yaml` to add the relevant port numbers and backend services.

## How to insert/get secrets into HashiCorp Vault

1.  Insert secrets
   1.  Using POST request
      1. ```bash
            curl --header "X-Vault-Token: <ROOT_TOKEN_VALUE>" --request POST \
            --data '{"data": {"key":{"auths":{"https://index.docker.io/v1/":{"auth":"<BASE64-ENCODED_USERNAME:PASSWORD>","email": "<EMAIL>","password": "<PASSWORD>","username": "<USERNAME>"}}}}}' \
            https://vault.mydomain.com/v1/secret/data/dockerhub-secret
         ```
   2.  Using CLI
      1.  `kubectl exec -it -n hashicorp-vault hashicorp-vault-0 -- sh`
      2.  `export VAULT_TOKEN=<ROOT_TOKEN_VALUE>`
      3.  `vault kv put -mount=secret <secret-name> <key1>=<secret-value1> <key2>=<secret-value2>`
      4.  ```vault kv put -mount=secret <secret-name> `cat .env` ```
2. Get secrets
   1. Using CLI
      1.  `kubectl exec -it -n hashicorp-vault hashicorp-vault-0 -- sh`
      2.  `export VAULT_TOKEN=<ROOT_TOKEN_VALUE>`
      3.  `vault kv get -mount=secret <secret-name>`
