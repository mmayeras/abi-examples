## Requirements

A linux machines to run the playbooks and the oc clients

1. Openshift-install client
```
curl -LO https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-install-linux.tar.gz
```

2. Openshift client
```
curl -LO https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz
```

3. dellemc.openmanage Ansible collection
```
$ ansible-galaxy collection install dellemc.openmanage
```

4. nmstatectl (Fedora updates)
```
$ dnf install nmstate -y
```



## LAB Installation

The cluster is deployed using the agent-based installation method.  
https://docs.openshift.com/container-platform/4.17/installing/installing_with_agent_based_installer/preparing-to-install-with-agent-based-installer.html#installation-bare-metal-agent-installer-config-yaml_preparing-to-install-with-agent-based-installer

# Table of Contents

1. [Create install-config.yaml an agent-config.yaml](#createfiles)
2. [Generate the ISO](#iso)
3. [Copy the ISO in the helper host httpd server](#uploadiso)
4. [Mount the remote share on iDRACs](#idrac)
5. [Monitor install](#mon)

6. [Day-2 - Add new nodes](#day2)


### 1. Create install-config.yaml an agent-config.yaml <a name="createfiles"></a>

The repository contains all the playbooks and variables

[vars.yaml](vars.yaml) contains all the hostnames, IPs, MAC addresses, iDRAC IPs 

a. Review and/or edit [vars.yaml](vars.yaml)


b. Generate the yaml files using the playbooks

The files are generated based on the Jinja templates [install-config.yaml.j2](install-config.yaml.j2) and [agent-config.yaml.j2](agent-config.yaml.j2)

```
$ ansible-playbook generate_install-config.yaml
$ ansible-playbook generate_agent-config.yaml
```

c. The installer consumes the files when we generate the iso, backup the files using 
```
$  ansible-playbook  backup_installation_files.yaml
```

d. Create a directory for the installation and copy the files into it
```
$ mkdir install_dir && mv install-config.yaml agent-config.yaml install_dir/
```

### 2. Generate the ISO <a name="iso"></a>
```
$  ./openshift-install agent create image --dir install_dir/
```

### 3. Upload the ISO on the helper node <a name="uploadiso"></a>
```
$ ansible-playbook scp_agent_iso.yaml
```

This is a simple scp command, you can do it manually.

### 4. Mount the remote share on iDRACs and configure the nodes boot order, the node will automatically reboot <a name="idrac"></a>
```
$ ansible-playbook idrac_remote_mount.yaml
```

### 5. Monitor the installation process <a name="mon"></a>

```
$ ./openshift-install agent wait-for bootstrap-complete   --log-level=debug  --dir install_dir/
$ ./openshift-install agent wait-for install-complete     --log-level=debug  --dir install_dir/
```

### 6. Day-2 - Add new nodes <a name="day2"></a>

Requires rsync installed on the machine.

https://docs.openshift.com/container-platform/4.17/nodes/nodes/nodes-nodes-adding-node-iso.html  

Create a nodes-config.yaml file using same format as agent-config.yaml


Then, build the iso

````
$ oc adm node-image create nodes-config.yaml
````

Then use the node.x86_64.iso generated to boot the machine(s)

Once the node is ready, you'll add to manually approve the csr

```
$ oc get csr
csr-lcv42   5m7s   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   <none>              Pending
oc adm certificate approve csr-lcv42
```

And the node will join the cluster

```
$ oc get nodes
...
worker-n.abi.example.com    Ready   worker                 7s    v1.30.7
...
```