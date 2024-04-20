# ACM Policies for Openshift

The goal of this project is to configure an Openshift cluster in a fully automated way with the less possible manual intervention.

The starting point for this configuration is a fully deployed OCP 4 cluster.

## ACM installation

Before ACM policies can be used to configure the OCP cluster, the RHACM operator must be installed and configured. Ansible is used for this tak.

**Ansible installation** 

To intall ansible core components use the following command in the host where the playbooks are going to be run:
```
sudo python3 -m pip install ansible
```
Check the ansible modules and check if the kubernetes.core collection is installed:

```
ansible-galaxy collection list
...
kubernetes.core               1.2.1 
...
```
If the [version is old](https://galaxy.ansible.com/ui/repo/published/kubernetes/core/), like in this case, update the kubernetes core andible modules, run this command as the user who will run the playbooks:
```
ansible-galaxy collection install kubernetes.core --upgrade
```
Verify that the kubernetes core modules have been updated
```
ansible-galaxy collection list

...
# /home/lab-user/.ansible/collections/ansible_collections
Collection      Version
--------------- -------
kubernetes.core 3.0.1
```
Install the kubernetes python libraries:
```
sudo python3 -m pip install kubernetes
```

**Prepare for playbook execution**

In order for ansible to stablish a secure connections with the Openshift API, the k8s ansible modules need access to the root CA certificate used by the cluster's API service. Obtain the CA bundle running the following command.

Make sure the resulting file is stored in the Ansible directory:

```
$ oc rsh -n openshift-authentication <oauth-openshift-pod> cat /run/secrets/kubernetes.io/serviceaccount/ca.crt > Ansible/api-ca.crt
```
The playbook requires cluster admin credentials so it can make configuration changes to the cluster. The credentials are expected in the file **Ansible/group_vars/user_credentials.vault**.

The credentials must be assigned to the variable **token**, its value can be obtained using the following command:  
```
$ oc whoami -t
sha256~roxev5_y0w4o-VjSadl3tiTkXqSOVhCRMvmV-K3xpfw
```
The contents of the **Ansible/group_vars/user_credentials.vault** file should look like:
```
token: sha256~roxev5_y0w4o-VjSadl3tiTkXqSOVhCRMvmV-K3xpfw
```
The above file should be encrypted with ansible-vault.  Make sure to generate the vault-id file in the Ansible directory:
```
$ cd Ansible
$ echo "token: $(oc whoami -t)" > group_vars/user_credentials.vault
$ pwmake 128 > vault-id
$ ansible-vault encrypt --vault-id vault-id group_vars/user_credentials.vault
```
The vault-id file is passed as an argument to the playbook later.

The playbook needs the API entrypoint of the Openshift cluter. Assign the value to the ansible variable **api_entrypoint**. The value can be obtained running the following command:
```
$ oc whoami --show-server
https://api.cluster-lh48t.lh48t.sandbox180.opentlc.com:6443
```

Define the RHACM operator channel and version.  For the channel, the default value is defined in the variable **rhacm-subs-channel**, in the file **Ansible/roles/rhacm_install/defaults/main.yml**.  For the version, the value is defined in the variable **rhacm-subs-version** defined in the same file as before.

The channel and version information can be obtained using the oc mirror plugin:
```
$ oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.14 --package=advanced-cluster-management
NAME                         DISPLAY NAME                                DEFAULT CHANNEL
advanced-cluster-management  Advanced Cluster Management for Kubernetes  release-2.10

PACKAGE                      CHANNEL       HEAD
advanced-cluster-management  release-2.10  advanced-cluster-management.v2.10.1
advanced-cluster-management  release-2.8   advanced-cluster-management.v2.8.6
advanced-cluster-management  release-2.9   advanced-cluster-management.v2.9.3

$ oc mirror list operators --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.14 --package=advanced-cluster-management --channel=release-2.10
VERSIONS
2.10.0
2.10.1
```
For example, to install version 2.10.1 the following values should be used:
```
rhacm_subs_channel: release-2.10
rhacm_subs_version: advanced-cluster-management.v2.10.0
```

Run the ansible playbook, make sure to assing the values obtained in the steps before for the variables: api_entrypoint and api_ca_cert.  And reference the **vault-id** file.
```
ansible-playbook -vvv ocp-initialization.yaml -e api_entrypoint="https://api.cluster-bhj4z.bhj4z.sandbox1490.opentlc.com:6443" -e api_ca_cert=api-ca.crt --vault-id vault-id
```



