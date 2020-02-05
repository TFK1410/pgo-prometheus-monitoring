# pgo-prometheus-monitoring
This a bunch of yaml bundles and steps which can be used to setup prometheus monitoring of the postgres operator by CrunchyData.

The setup will closely replicate what it looks like with the openshift cluster monitoring operator. The setup has been tested on OpenShift 3.11.
The setup contains:
* Prometheus-operator
* Prometheus
* Alertmanager
* Grafana
* netapp-api-exporter (optional)

The steps for deployment are as follows:
## Basic project setup
1. Create pgo-monitoring project
1. Create CRDs if necessary
1. Deploy bundle.yml in the pgo-monitoring namespace

## Grafana password setup
1. Generate plain password file and keep it safe. This will be used to access prometheus using the internal user by grafana:
   ```bash
   # head /dev/urandom | tr -dc A-Za-z0-9 | head -c255 > plain_password
   ```
1. Generate htpasswd using this password:
   ```bash
   # htpasswd -cbs internal.htpasswd internal $(cat plain_password)
   Adding password for user internal
   ```
1. Create the secret in the project from the file:
   ```bash
   # oc -n pgo-monitoring create secret generic --from-file=auth=internal.htpasswd prometheus-k8s-htpasswd
   ```

## Deploy Prometheus along with basic servicemonitors
1. Create oauth session secret:
   ```bash
   # oc -n pgo-monitoring create secret generic prometheus-k8s-pgo-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
1. Replace the "<CLUSTER>" placeholders in the prometheus.yml file with the domain name of the cluster
1. Deploy the prometheus.yml bundle
1. Deploy the servicemonitors.yml

## Deploy Alertmanager
1. Create oauth session secret:
   ```bash
   # oc -n pgo-monitoring create secret generic alertmanager-pgo-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
1. Replace the "<CLUSTER>" placeholders in the alertmanager.yml file with the domain name of the cluster
1. Deploy the alertmanager.yml

## Deploy Grafana secrets
1. Create oauth session secret:
   ```bash
   # oc -n pgo-monitoring create secret generic grafana-pgo-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
1. Edit the grafana-datasource.yml file and add the previously created plain password as the value for basicAuthPassword
1. Create a secret file out of the modified grafana-datasource.yml
   ```bash
   # oc -n pgo-monitoring create secret generic --from-file=prometheus.yml=grafana-datasource.yml grafana-pgo-datasources
   ```
## Deploy Grafana dashboards
1. Create dashboards, each in a single configmap, from the files in the dashboards directory 
   (a single configmap would be too big, add additional dashboards as you see fit):
   ```bash
   # oc -n pgo-monitoring create configmap grafana-pgo-dashboard-bloat --from-file=dashboards/Bloat_Details.json
   # oc -n pgo-monitoring create configmap grafana-pgo-dashboard-crud --from-file=dashboards/CRUD_Details.json
   # oc -n pgo-monitoring create configmap grafana-pgo-dashboard-yml --from-file=dashboards/crunchy_grafana_dashboards.yml
   # oc -n pgo-monitoring create configmap grafana-pgo-dashboard-fs --from-file=dashboards/Filesystem_Details.json
   # oc -n pgo-monitoring create configmap grafana-pgo-dashboard-pgbackrest --from-file=dashboards/PGBackrest.json
   # oc -n pgo-monitoring create configmap grafana-pgo-dashboard-pgdetails --from-file=dashboards/PG_Details.json
   # oc -n pgo-monitoring create configmap grafana-pgo-dashboard-pgoverview --from-file=dashboards/PG_Overview.json
   # oc -n pgo-monitoring create configmap grafana-pgo-dashboard-tablesize --from-file=dashboards/TableSize_Details.json
   ```
All of those dashboards were downloaded from the pgmonitor project at v3.2.

## Deploy Grafana
1. Create a secret out of the grafana.ini file:
   ```bash
   # oc -n pgo-monitoring create secret generic --from-file=grafana.ini grafana-pgo-config
   ```
1. Deploy grafana.yml (log in and make sure that Prometheus is set as the default datasource)

## Deploy Prometheus rules
1. Deploy prometheusrules.yml
1. Deploy prometheusrecs.yml for some additional recording rules

## Deploy netapp-api-exporter (optional)
1. Build the netapp-api-exporter image from the provided Dockerfile
   ```bash
   # oc project pgo-monitoring
   # cat netapp-api-exporter/Dockerfile | oc new-build --name netapp-api-exporter -D -
   ```
1. Create your own netapp_filers.yaml file from the template provided in the netapp-api-exporter directory
1. Create a secret file out of the modified netapp_filers.yaml file
   ```bash
   # oc -n pgo-monitoring create secret generic --from-file=netapp_filers.yaml=netapp_filers.yaml netappfilers
   ```
1. Deploy netapp-api-exporter.yml
