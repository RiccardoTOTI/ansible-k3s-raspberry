# ansible-k3s-raspberry

# Prerequisites
Install the Raspbian OS (64-bit) on your Raspberry Pies, connect them to your local network, give them a hostname, and enable their SSH servers. 
Install Ansible on your local machine. (I use the WSL 2 subsystem in Windows to run Ansible.)
Install kubectl.
Optional: Install Helm.

## Ansible setup and components
To provision our cluster with ansible we need an inventory file, playbook file(s), and some initial SSH setup since ansible uses SSH to connect to nodes.

Inventory file
In Ansible, the inventory file is a file that contains a list of target hosts or nodes that Ansible can manage. In this case, it will contain all our Raspberry Pis.

```yaml
workers:
  hosts:
    nerminworker1:
      ansible_host: ip_address
      ansible_user: user

masters:
  hosts:

# This hostgroup is designed to only contains the initial bootstrap master node
bootstrapMaster:
  hosts:
    nerminmaster:
      ansible_host: master_ip_address
      ansible_user: user

pies:
  children:
    bootstrapMaster:
    masters:
    workers:
  vars:
    k3s_version: v1.24.10+k3s1
```
We have four groups in our setup:

bootstrapMaster: This is the first node that will get a k3s server and will create the token used by other nodes to register with the cluster.

- **masters** : These are the remaining k3s server nodes that will register themselves via the bootstrapMaster.
- **workers** : These are the k3s worker/agent nodes that will run workloads in the cluster. They also register - themselves via the bootstrapMaster and the generated k3s token.
pies: This group contains all the masters and workers combined.

# SSH connections (Optional, skip to [Install without key](https://github.com/RiccardoTOTI/ansible-k3s-raspberry/edit/main/README.md#installation-without-ssh-key)
Add your public SSH key to each Raspberry Pi to avoid being prompted for passwords when creating sessions.

`$ ssh-copy-id <user>@<host>`
In this specific case:

- `$ ssh-copy-id user@master_ip_address`
- `$ ssh-copy-id user@ip_address`
You can also use the hostname instead of the IP.

Note: Make sure you have generated an SSH key pair in your WSL2 system. You can check by running:

`$ ls ~/.ssh/id_*.pub`
If you don't see any files listed, you'll need to generate an SSH key pair. You can do this by running:

`$ ssh-keygen`
Follow the prompts to generate a new key pair. Make sure to keep your private key secure and do not share it with others.

# Install Playbook
The install-k3s-playbook.yaml file contains plays for installing k3s masters and workers on the Raspberry Pis. This includes enabling memory cgroups as required by the documentation. The playbook also retrieves the kubeconfig file from the bootstrap node and places it in the current directory with the name k3sconfig.

## Standard Installation

To create the k3s cluster, run the install playbook:

```bash
ansible-playbook -i inventory.yaml install.yaml
```

## Installation Without SSH Key

If you haven't set up SSH keys, you can use password authentication:

```bash
ansible-playbook -i inventory.yaml install.yaml --ask-pass
```

This command will prompt you for the SSH password for the hosts.

> **Note:** Password authentication is not recommended for security reasons. It may be acceptable for small home environments where all hosts share the same password, but using SSH keys is strongly advised for better security.

---

The playbook downloaded the kubeconfig file from the bootstrap master to the current directory on the local machine with the name k3sconfig. The file looks like this:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://127.0.0.1:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: ...
    client-key-data: ...
```

Replace the localhost IP in the server value with the IP or hostname of the bootstrap master node.

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://nerminmaster:6443  # <----- This value here.
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: ...
    client-key-data: ...
```

Verify that the cluster is up and running:

```bash
$ kubectl get nodes --kubeconfig k3sconfig
NAME            STATUS   ROLES                       AGE     VERSION
nerminmaster    Ready    control-plane,etcd,master   8m42s   v1.24.10+k3s1
nerminworker1   Ready    <none>                      8m7s    v1.24.10+k3s1
```

# Documentation 

- https://github.com/hatati/PiClusterChef/tree/master
- https://dev.to/hatati/cook-up-a-k3s-cluster-on-raspberry-pies-with-ansible-4bb4
