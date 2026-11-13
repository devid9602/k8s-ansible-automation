# Kubernetes and Ansible SSH Lab

A small lab that uses Kubernetes Pods as SSH managed hosts for Ansible practice. It includes a Pod and NodePort Service for an Ubuntu-based host, an Ansible inventory example, and a playbook that creates a Kubernetes namespace.

## Repository contents

- `ansible_host_pod.yml`: creates the `ansible-host-2` Pod and exposes SSH through NodePort `32223`.
- `inventory.ini`: example Ansible targets. Replace its sample address and ports with endpoints reachable from your Ansible controller.
- `create_namespace.yml`: creates the `ansible-demo` namespace using the `community.kubernetes` collection.

The Pod reads its SSH password from the `ansible-host-credentials` Kubernetes Secret. The manifest currently runs the container as root with a privileged security context, installs packages at startup, and enables password-based SSH. Treat this as a disposable learning lab, not a production pattern.

## Prerequisites

- A Kubernetes cluster and a working `kubectl` context.
- Ansible on a machine that can reach the cluster's NodePort.
- The Ansible collection used by the namespace playbook:
  `ansible-galaxy collection install community.kubernetes`.

## Run the lab

Create the namespace and a Secret containing the lab password. Supply the password through a local environment variable; do not commit it to this repository.

```sh
kubectl create namespace ansible-demo
read -rsp "Ansible host password: " ANSIBLE_PASSWORD
echo
kubectl create secret generic ansible-host-credentials \
  --namespace ansible-demo \
  --from-literal=password="$ANSIBLE_PASSWORD"
unset ANSIBLE_PASSWORD
```

Apply the host manifest into that namespace:

```sh
kubectl apply --namespace ansible-demo -f ansible_host_pod.yml
kubectl get pods,services --namespace ansible-demo
```

Update `inventory.ini` to use the cluster node address reachable from the Ansible controller and the Service's NodePort. The checked-in inventory contains sample endpoints; the manifest defines the `host2` target on port `32223`. Then test connectivity:

```sh
ansible -i inventory.ini all -m ping
```

To run the namespace playbook instead of creating the namespace with `kubectl`, use:

```sh
ansible-playbook create_namespace.yml
```

## Security and cleanup

Do not expose the NodePort to the public internet. Restrict access to the Ansible controller, use throwaway credentials, and remove the lab when finished:

```sh
kubectl delete namespace ansible-demo
```

The Pod's root and privileged settings, password-based SSH, and package installation on every start are intentionally simple for experimentation. Replace these with least-privilege identities, hardened images, and an approved secret-management approach before adapting the pattern for real workloads.
