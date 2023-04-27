# Runbook

## Requirements
- Install Ansible : https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html
- Install k3sup on your laptop : https://github.com/alexellis/k3sup
- Install kubectx et kubens : https://github.com/ahmetb/kubectx
- Install helm on your laptop: https://helm.sh/docs/intro/install/

### Context

- **6 VMs**
    - 3 Master nodes 
    - 3 Worker nodes

- **Resources per node**
    - 2 VCPUs
    - 4 Gio RAM
    - 50 Gio Disk

- **OS**
    - Debian 11 - 'Bullseye' 

---

#### Provisionning the cluster:
`$ cd ansible-k3s-cluster && ansible-playbook -i inventory.ini main.yml`

---

#### Configuring the cluster
- Add Flannel CNI
*For some reason Flannel is not embedded by default with k3sup*
`$ kubectl create -f ../kube/kube-flannel.yaml`

If ever the flannel pods crashloopbackoff check the logs, if it's a podCIDR problem check your address ranges and modify the kube-flannel.yml file accordingly in the net-conf.json key
```
- Get podCIDR
$ kubectl get node k3s-<node> -o=jsonpath='{.spec.podCIDR}'

- Modification of the kube-flannel file, example:
  net-conf.json: |
    {
      "Network": "10.42.4.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }

- reload of configuration:
$ kubectl delete -f ../kube/kube-flannel.yaml 
$ kubectl create -f ../kube/kube-flannel.yaml 
```
- Force restart of CoreDNS
`$ kubectl -n kube-system delete pod -l k8s-app=kube-dns`

- Prevent pod scheduling on masters
`$ kubectl taint nodes k3s-master-node-1 k3s-master-node-2 k3s-master-node-3 node-type=master:NoSchedule`

- Create new NS
`$ kubectl create ns monitoring`

---

#### Install Grafana
- Adding the Grafana repository to Helm : 
`$ helm repo add grafana https://grafana.github.io/helm-charts`

- Update helm repository: 
`$ helm repo update`

- Installing Grafana:
`$ helm upgrade --install --namespace monitoring grafana grafana/grafana -f helm-charts/grafana/values.yaml`

- Get Credentials:
`$ kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`

- Create a port-forward to access Grafana locally
```
$ export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")

$ kubectl --namespace monitoring port-forward $POD_NAME 3000

```
- Access to Grafana:
`localhost:3000`

---

#### Install Victoria Metrics

Adding the Grafana repository to Helm: \
`$ helm repo add victoria-metrics https://victoriametrics.github.io/helm-charts/` 

Update helm repository : \
`$ helm repo update` 

Installing Victoria Metrics Cluster:\
`$ helm upgrade --install --namespace monitoring --create-namespace victoria-metrics victoria-metrics/victoria-metrics-cluster -f .helm-charts/victoria-metrics/values.yml`

---

#### Adding the datasource in grafana
In grafana add a new datasource of type Prometheus, rename it Victoria Metrics and give it this URI:
`http://victoria-metrics-victoria-metrics-cluster-vmselect.monitoring.svc.cluster.local:8481/select/0/prometheus/`

---

#### Install VM-Agent
`helm upgrade --install --namespace monitoring vm-agent victoria-metrics/vmagent -f helm-charts/vmagent/values.yml`

---
*Start operating !* 🚀 