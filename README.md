# Metal3 Lab

## Command Cheat-Sheet

- `kubectl get hostfirmwarecomponents <node name> -n baremetal-operator-system -o yaml` - Gets the  list of
  firmware available for the node.
- `kubectl get bmh <node name> -n baremetal-operator-system` - Get the current status of the node.
- `kubectl describe bmh <node name> -n baremetal-operator-system` - Get the full description of the node.
- `kubectl annotate bmh <node name> reboot.metal3.io="" -n baremetal-operator-system` - Reboots the node, handy for firmware upgrades.

## Overview

### Services:

k3s - Single node lightweight kubernetes for hosting rest of services.
metal3 - Baremetal provisioning service

## Deploying Metal3:

1. Install k3s on control node: `curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -`
1. Setup cert-manager on control node: `kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml`
1. Install IrSO (Ironic Standalone Operator): `kubectl apply -f https://github.com/metal3-io/ironic-standalone-operator/releases/latest/download/install.yaml`
1. Create the baremetal-operator-system namespace `kubectl create namespace baremetal-operator-system`
1. Customize the yaml below and apply it with `kubectl -f apply install-ironic.yaml`
1. Install BMO (Bare Metal Operator): `kubectl apply -k bmo/`

## Deploy Let's Encrypt for Cert Manager

1. Modify and apply the Let's Encrypt ClusterIssuer `kubectl apply -f oci/cluster-issuer.yaml`

## Deploy Zot as OCI registry for storing OS images:

1. Add the Zot help repo:
  ``` bash
  helm repo add project-zot http://zotregistry.dev/helm-charts
  helm repo update project-zot
  ```
1. If you haven't already, ensure the file descripters are set high:
  ``` bash
  echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
  echo "fs.inotify.max_user_instances=8192" | sudo tee -a /etc/sysctl.conf

  # Reload the configuration
  sudo sysctl -p
  ```
1. Create zot namespace `kubectl create namespace zot-system`
1. Setup basic http auth for pushing to Zot (pulling is no-auth)
  ``` bash
  htpasswd -Bc auth admin
  kubectl create secret generic zot-auth-secret --from-file=htpasswd=auth -n zot-system
  ```
1. Customize the `zot-values.yaml` file and install it:
  ``` bash
  helm upgrade zot-registry project-zot/zot \
  --namespace zot-system \
  --create-namespace \
  -f oci/values-zot.yaml
  ```
1. Apply the servicemonitor manually due to bug in helm chart `kubectl apply -f oci/servicemonitor-zot.yaml`
1. Modify and apply the Let's Encrypt ingress for zot: `kubectl apply -f oci/le-ingress-zot.yaml`

## Install Monitoring

1. Create the monitoring namespace with `kubectl create namespace monitoring`
1. Add the helm repos:
    ``` bash
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo add sloth https://slok.github.io/sloth
    helm repo update
    ```
1. Install the various monitoring charts:
    ``` bash
    # Core metrics
    helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      --create-namespace \
      -f monitoring/values-monitoring.yaml

    # Loki
    helm upgrade --install loki grafana/loki \
      --namespace monitoring \
      -f monitoring/values-loki.yaml

    # Alloy (log collector)
    helm upgrade --install alloy grafana/alloy \
      --namespace monitoring \
      -f monitoring/values-alloy.yaml

    # Tempo DB (tracing)
    helm upgrade --install tempo grafana/tempo \
      --namespace monitoring \
      -f monitoring/values-tempo.yaml

    # Beyla (eBPF based tracing)
    helm upgrade --install beyla grafana/beyla \
      --namespace monitoring \
      -f monitoring/values-beyla.yaml

    # SLOth (SLO monitoring)
    helm install sloth sloth/sloth \
      --namespace monitoring \
      --create-namespace

    # SLOth Demo SLO (availability of ironic)
    kubectl apply -f monitoring/sloth-demo-slo.yaml
    ```
1. Disable Loki-canary if you want (it generates a lot of data) `kubectl delete daemonset loki-canary -n monitoring`

1. Install the monitoring for the BMCs. This uses `mrlhansen/idrac_exporter` and is compatible with most
  redfish based BMCs (iDRAC, iLO, SMC). Our configuration is just hand configured YAML because of the low
  number of servers - in prod you'd want to have some automation (helm templating) around generating the
  config. This configuration also re-uses the metal3 baremetalhost bmc secret to reduce duplication.
  ``` bash
  kubectl apply -f idrac_prometheus.yaml
  ```

### Grafana Dashboards:

1. For SLOth, `14643` is for the high-level SLOs and `14348` is for the detailed SLO view.
1. For BMCs there are two from `mrlhansen/idrac_exporter`:
   ```
   # Overview
   https://github.com/mrlhansen/idrac_exporter/blob/master/grafana/idrac_overview.json

   # Per Node
   https://github.com/mrlhansen/idrac_exporter/blob/master/grafana/idrac.json
   ```

### Handy Zot CLI Commands:

``` bash
# Get the application URL by running these commands:
export NODE_PORT=$(kubectl get --namespace zot-system -o jsonpath="{.spec.ports[0].nodePort}" services zot-registry)
export NODE_IP=$(kubectl get nodes --namespace zot-system -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT

# Check Zot metrics are going out by forwarding port
kubectl port-forward svc/zot-registry -n zot-system 5000:5000
curl http://localhost:5000/metrics

# Check the status of your Helm release
helm list -n zot-system

# Find the exact NodePort assigned to your service
export NODE_PORT=$(kubectl get --namespace zot-system -o jsonpath="{.spec.ports[0].nodePort}" services zot-registry-zot)
export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")

# Verify the registry is responding to API calls (should return an empty repository list)
curl http://$NODE_IP:$NODE_PORT/v2/_catalog

# Get IP/Port of Zot
kubectl get svc -n zot-system

# Login to using oras
oras login <k8s hostname> -u admin -p <zot password>

# Example of using Oras to push artifact:
touch esxi-custom-7.0.iso
oras push <k8s hostname>:31234/vmware-operator/esxi:7.0-custom \
  esxi-custom-7.0.iso:application/vnd.acme.vmware.iso.layer.v1+tar
```

### Handy Monitoring CLI Commands:

``` bash
# Get Password:
kubectl --namespace monitoring get secrets monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# Check status:
kubectl --namespace monitoring get pods -l "release=monitoring"

# Access local instance:
export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitoring" -oname)
  kubectl --namespace monitoring port-forward $POD_NAME 3000
```

## Adding Servers

Adding servers requires configuration of a multi-part YAML file that includes
the BMC credentials, static network configuration and other information.

  CAUTION: Be careful not to commit any secrets to the repository!!!

1. Customize the file `server-bmh.ymal` with the appropriate settings.
1. Apply the file using `kubectl-server-bmh.yaml`

## Provisioning Servers

1. Upload your image to the images folder created earlier.
1. Configure the userdata secret with the relevant information add apply it. `user-data-secret.yaml`
1. Update your server's bmh file with the appropriate secret name and uncomment
the image + userdata fields to trigger provisioning.

## Updating Firmware

1. Upload the firmware to the images folder created earlier.
1. Configure `firmware-update.yaml` or reference the URL in the comment on that file.
1. Apply the firmware file and wait for whichever condition to trigger the firmware update.


