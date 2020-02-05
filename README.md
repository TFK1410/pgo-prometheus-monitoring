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
2. Create CRDs if necessary
3. Deploy bundle.yml in the pgo-monitoring namespace

## Grafana password setup
4. Generate plain password file and keep it safe. This will be used to access prometheus using the internal user by grafana:
   ```bash
   # head /dev/urandom | tr -dc A-Za-z0-9 | head -c255 > plain_password
   ```
5. Generate htpasswd using this password:
   ```bash
   # htpasswd -cbs internal.htpasswd internal $(cat plain_password)
   Adding password for user internal
   ```
6. Create the secret in the project from the file:
   ```bash
   # oc -n pgo-monitoring create secret generic --from-file=auth=internal.htpasswd prometheus-k8s-htpasswd
   ```

## Deploy Prometheus along with basic servicemonitors
7. Create oauth session secret:
   ```bash
   # oc -n pgo-monitoring create secret generic prometheus-k8s-pgo-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
8. Deploy the prometheus.yml bundle
9. Deploy the servicemonitors.yml

## Deploy alertmanager
10. Create oauth session secret:
   ```bash
   # oc -n pgo-monitoring create secret generic alertmanager-pgo-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
11. Deploy the alertmanager.yml

## Deploy Grafana secrets
12. Create oauth session secret:
   ```bash
   # oc -n pgo-monitoring create secret generic grafana-pgo-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
13. Edit the grafana-datasource.yml file and add the previously created plain password as the value for basicAuthPassword
14. Create a secret file out of the modified grafana-datasource.yml
   ```bash
   # oc -n pgo-monitoring create secret generic --from-file=prometheus.yml=grafana-datasource.yml grafana-pgo-datasources
   ```
## Deploy Grafana dashboards
15. Create dashboards, each in a single configmap, from the files in the dashboards directory 
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
16. Create a secret out of the grafana.ini file:
   ```bash
   # oc -n pgo-monitoring create secret generic --from-file=grafana.ini grafana-pgo-config
   ```
17. Deploy grafana.yml (log in and make sure that Prometheus is set as the default datasource)

## Deploy additional Prometheus rules
18. Deploy prometheusrules.yml
19. Deploy prometheusrecs.yml for some additional recording rules

## Deploy netapp-api-exporter (optional)
20. Build the netapp-api-exporter image from the provided Dockerfile
   ```bash
   # oc project pgo-monitoring
   # cat netapp-api-exporter/Dockerfile | oc new-build --name netapp-api-exporter -D -
   ```
21. Create your own netapp_filers.yaml file from the template provided in the netapp-api-exporter directory
22. Create a secret file out of the modified netapp_filers.yaml file
   ```bash
   # oc -n pgo-monitoring create secret generic --from-file=netapp_filers.yaml=netapp_filers.yaml netappfilers
   ```
23. Deploy netapp-api-exporter.yml
