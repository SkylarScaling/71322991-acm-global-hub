Example inventory:

```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
  vars:
    home_dir: <home_dir>
    tmp_dir: <tmp_dir>
    openshift_version: "4.20"
    ocp_patch_version: "13"
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
      workers:
        replicas: "3"
      ssh_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
      pull_secret: "{{ lookup('file', '~/pull-secret.json') }}"
```