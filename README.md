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
1. Configure a local web server for hosting OS images:
  ```
  docker run -d \
    --name simple-images \
    -p 8080:80 \
    -v /full/path/to/image/folder:/usr/share/nginx/html:ro \
    --restart unless-stopped \
    nginx:alpine
  ```
1. Install IrSO (Ironic Standalone Operator): `kubectl apply -f https://github.com/metal3-io/ironic-standalone-operator/releases/latest/download/install.yaml`
1. Create the baremetal-operator-system namespace `kubectl create namespace baremetal-operator-system`
1. Customize the yaml below and apply it with `kubectl -f apply install-ironic.yaml`
1. Install BMO (Bare Metal Operator): `kubectl apply -k bmo/`

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


