# pgo-prometheus-monitoring
This a bunch of yaml bundles and steps which can be used to setup prometheus monitoring. These examples are based around monitoring of the postgres operator by CrunchyData and its postgres clusters.

The setup has been tested on OpenShift 3.11. It contains:
* Prometheus-operator
* Prometheus
* Alertmanager
* Grafana
* netapp-api-exporter (optional)
* kube-state-metrics (optional)
* servicenow-receiver (optional)

The steps for deployment are as follows:
## Basic project setup
1. Create monitoring project
1. Deploy prometheus-opeartor/bundle.yml in the monitoring namespace
1. Create CRDs if necessary

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
   # oc -n custom-monitoring create secret generic --from-file=auth=internal.htpasswd prometheus-k8s-htpasswd
   ```

## Deploy Prometheus along with basic servicemonitors
1. Create oauth session secret:
   ```bash
   # oc -n custom-monitoring create secret generic prometheus-k8s-custom-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
1. Replace the "CLUSTER" placeholders in the prometheus.yml file with the domain name of the cluster
1. Deploy the prometheus/prometheus.yml bundle
1. Deploy the prometheus/servicemonitors.yml. 

This contains a few basic ServiceMonitor objects. This also includes a monitor for the crunchy-collect containers.

## Deploy Alertmanager
1. Create oauth session secret:
   ```bash
   # oc -n custom-monitoring create secret generic alertmanager-custom-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
1. Replace the "CLUSTER" placeholders in the alertmanager.yml file with the domain name of the cluster
1. Deploy the alertmanager/alertmanager.yml

The alertmanager bundle contains the secret with an example configuration for the receivers. I suggest adjusting that to your own needs.

## Deploy Grafana secrets
1. Create oauth session secret:
   ```bash
   # oc -n custom-monitoring create secret generic grafana-custom-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
1. Edit the grafana/grafana-datasource.yml file and add the previously created plain password as the value for basicAuthPassword
1. Create a secret file out of the modified grafana-datasource.yml
   ```bash
   # oc -n custom-monitoring create secret generic --from-file=prometheus.yml=grafana/grafana-datasource.yml grafana-custom-datasources
   ```
## Deploy Grafana dashboards for pg cluster monitoring
1. Create dashboards, each in a single configmap, from the files in the dashboards directory 
   (a single configmap would be too big, add additional dashboards as you see fit):
   ```bash
   # oc -n custom-monitoring create configmap grafana-pgo-dashboard-bloat --from-file=grafana/dashboards/Bloat_Details.json
   # oc -n custom-monitoring create configmap grafana-pgo-dashboard-crud --from-file=grafana/dashboards/CRUD_Details.json
   # oc -n custom-monitoring create configmap grafana-pgo-dashboard-yml --from-file=grafana/dashboards/crunchy_grafana_dashboards.yml
   # oc -n custom-monitoring create configmap grafana-pgo-dashboard-fs --from-file=grafana/dashboards/Filesystem_Details.json
   # oc -n custom-monitoring create configmap grafana-pgo-dashboard-pgbackrest --from-file=grafana/dashboards/PGBackrest.json
   # oc -n custom-monitoring create configmap grafana-pgo-dashboard-pgdetails --from-file=grafana/dashboards/PG_Details.json
   # oc -n custom-monitoring create configmap grafana-pgo-dashboard-pgoverview --from-file=grafana/dashboards/PG_Overview.json
   # oc -n custom-monitoring create configmap grafana-pgo-dashboard-tablesize --from-file=grafana/dashboards/TableSize_Details.json
   ```
All of those dashboards were downloaded from the pgmonitor project at v3.2.

## Deploy Grafana
1. Create a secret out of the grafana.ini file:
   ```bash
   # oc -n custom-monitoring create secret generic --from-file=grafana/grafana.ini grafana-custom-config
   ```
1. Deploy grafana.yml (log in and make sure that Prometheus is set as the default datasource)

## Deploy Prometheus rules
1. Deploy prometheus/prometheusrules.yml
1. Deploy prometheus/prometheusrecs.yml for some additional recording rules

## Deploy netapp-api-exporter (optional)
This is a deployment for monitoring the storage usage of NetApp volumes from the server side. This is especially useful for the NFS volumes which shouldn't be monitored from the client side.

1. Build the netapp-api-exporter image from the provided Dockerfile
   ```bash
   # oc project custom-monitoring
   # cat netapp-api-exporter/Dockerfile | oc new-build --name netapp-api-exporter -D -
   ```
1. Create your own netapp_filers.yaml file from the template provided in the netapp-api-exporter directory
1. Create a secret file out of the modified netapp_filers.yaml file
   ```bash
   # oc -n custom-monitoring create secret generic --from-file=netapp_filers.yaml=netapp_filers.yaml netappfilers
   ```
1. Deploy netapp-api-exporter.yml

## Deploy kube-state-metrics (optional)
This is a separate kube-state-metrics deployment. I've created a separate one from the openshift-monitoring project as the one available there was outdated.

1. Insert the monitoring namespace name in the "NAMESPACE" markings in the bundle kube-state-metrics/kube-state-metrics.yml
1. Deploy the bundle kube-state-metrics/kube-state-metrics.yml

## Deploy servicenow-receiver (optional)
This is a separate servicenow-receiver deployment. This service can be used by AlertManager to send any incoming alerts as ServiceNow incidents. It is based on the https://github.com/fxinnovation/alertmanager-webhook-servicenow project.

1. Fill out the receiver configuration based on the example in the servicenow-receiver directory. The the servicenow webhook repository has more detailed instructions if required.
1. Insert the monitoring namespace name in the "NAMESPACE" markings in the bundle servicenow-receiver/servicenow-receiver.yml
1. Deploy the bundle servicenow-receiver/servicenow-receiver.yml