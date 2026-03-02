Example inventory:

```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
  vars:
    home_dir: <home_dir>
    tmp_dir: "{{ home_dir }}/tmp"
    openshift_version: "4.20"
    ocp_patch_version: "12"
    force_update: true
    ocp_cluster:
      name: <cluster_name>
      base_domain: <base_domain>
    aws:
      account_id: <aws_account_id>
      aws_access_key_id: <aws_access_key_id>
      aws_secret_access_key: <aws_secret_access_key>
      aws_region: <aws_region>
    install_config:
      cluster_name: "{{ ocp_cluster.name }}"
      base_domain: "{{ ocp_cluster.base_domain }}"
      control_plane:
        instance_type: "m6a.xlarge"
      workers:
        instance_type: "m6a.xlarge"
        replicas: "2"
      infra:
        instance_type: "m6a.4xlarge"
        replicas: "3"
      storage:
        instance_type: "m6a.4xlarge"
        replicas: "3"
      ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
      pull_secret: "{{ lookup('file', '~/pull-secret.json') | from_json }}"
```

---

### Inventory Variables

Before running the provisioning playbooks, you must populate the `<var_name>` placeholders in your inventory file. Here is a breakdown of what each section controls:

#### 📁 General Configuration

* **`home_dir`**: The absolute path to your local home directory (e.g., `/home/username`). This is used as the base path for downloads and configurations.
* **`openshift_version`** & **`ocp_patch_version`**: Dictates the exact OpenShift release to download and install (e.g., `4.20` and `12` installs `4.20.12`).
* **`force_update`**: Set to `true` if you want the playbook to overwrite your existing `oc` and `openshift-install` binaries with fresh downloads.

#### 🌐 Cluster Identity

* **`ocp_cluster.name`**: The unique name for your OpenShift cluster. This will be used to tag all AWS resources.
* **`ocp_cluster.base_domain`**: The Route53 DNS zone hosting your cluster (e.g., `sandbox.example.com`). Your final console URL will be `console-openshift-console.apps.<name>.<base_domain>`.

#### ☁️ AWS Credentials

* **`aws.account_id`**: Your 12-digit AWS Account ID.
* **`aws.aws_access_key_id`** & **`aws.aws_secret_access_key`**: An IAM user's programmatic access keys. This user needs sufficient privileges to provision VPCs, EC2 instances, Route53 records, and IAM roles.
* **`aws.aws_region`**: The AWS region to deploy into (e.g., `us-east-2`).

#### 🏗️ Node Architecture (`install_config`)

This block defines the specific EC2 instance types and replica counts for your OpenShift node topology:

* **`control_plane`**: The master nodes managing the cluster state (default: `m6a.xlarge`).
* **`workers`**: General-purpose worker nodes for standard application workloads.
* **`infra`**: Dedicated nodes for OpenShift ingress routers, image registry, and monitoring. Moving these off standard workers saves on subscription consumption.
* **`storage`**: Dedicated `m6a.4xlarge` heavy-compute nodes specifically tainted and labeled to house OpenShift Data Foundation (ODF) and your Ceph storage clusters.

#### 🔐 Secrets & Access

* **`ssh_key`**: Points to your local public SSH key (typically `~/.ssh/id_ed25519.pub`). This is injected into the EC2 instances so you can SSH into the nodes for emergency debugging.
* **`pull_secret`**: The path to your Red Hat pull secret (downloaded from `console.redhat.com`). This authenticates your cluster to pull core OpenShift and Operator images.

---
