# Running the AWS Demo Infrastructure Provisioning Playbook

This guide explains how to run `playbooks/provision_demo_infra.yml` to provision the AWS security groups, Red Hat Satellite 6.19 server (with RSA Let's Encrypt certificates), and RHEL target nodes (`rhel9-sat01.samboxaws.org` and `rhel10-sat01.samboxaws.org`) using the `aws_demo_infra` and `satellite_server_setup` roles.

## 1. Prerequisites & Repository Layout

Ensure your repository includes the playbook, role directories, and `ansible.cfg`:

```text
├── ansible.cfg
├── playbooks/
│   └── provision_demo_infra.yml
└── roles/
    ├── aws_demo_infra/
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    ├── satellite_server_setup/
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    └── aap_server_setup/
        ├── defaults/main.yml
        ├── tasks/main.yml
        └── templates/aap_inventory.j2
```

Required collections and credentials:
* **`amazon.aws`** collection available in your execution environment or control node.
* **AWS credentials** with permissions to create security groups and launch EC2 instances in your target region (default: `us-east-2`).
* **SSH private key** matching `key_name` so Ansible can connect to `ec2-user` on the newly launched RHEL instances.

## 2. Inventory Behavior (`add_host` vs. Static Inventory)

You **do not need a static inventory file** for `satellite_server` or `satellite_target_nodes` when launching instances through `aws_demo_infra`:

1. The first play runs on `localhost` (`connection: local`) and executes `aws_demo_infra`.
2. After launching the EC2 instances, `aws_demo_infra` uses `ansible.builtin.add_host` to dynamically populate the `satellite_server` and `satellite_target_nodes` groups in memory.
3. Subsequent plays in `playbooks/provision_demo_infra.yml` immediately target those in-memory groups.

*Note:* If you include a play targeting `aap_server` (`aap_server_setup`) and are not launching the AAP VM dynamically in that run, provide an inventory (or AAP inventory/host entry) that defines `aap_server`.

## 3. Prepare Your Variables (`vars.yml` & Ansible Vault)

Create a variables file (for example, `vars/demo_infra.yml`) with your AWS and Satellite values. Store sensitive values such as `satellite_admin_password` in an Ansible Vault file or pass them via extra vars / AAP credentials:

```yaml
---
aws_region: "us-east-2"
vpc_id: "vpc-0123456789abcdef0"
subnet_id: "subnet-0123456789abcdef0"
key_name: "your-aws-keypair-name"
admin_cidr: "0.0.0.0/0"
vpc_cidr: "10.0.0.0/16"

rhel9_ami_id: "ami-0123456789abcdef0"
rhel10_ami_id: "ami-0fedcba9876543210"

satellite_fqdn: "satellite.samboxaws.org"
satellite_instance_type: "m6i.2xlarge"
satellite_volume_size_gb: 150
satellite_swap_size_mb: 4096
satellite_org: "Sambox"
satellite_location: "AWS"
satellite_admin_user: "admin"
satellite_admin_password: "ChangeMeToAStrongPassword!"

enable_letsencrypt: true
letsencrypt_email: "admin@samboxaws.org"

target_node_instance_type: "t3.small"
target_node_swap_size_mb: 2048
satellite_activation_key_rhel9: "ak-rhel9-dev"
satellite_activation_key_rhel10: "ak-rhel10-dev"
```

## 4. Run from the Command Line

1. **Validate syntax and role resolution:**

   ```bash
   ansible-playbook -i /dev/null --syntax-check playbooks/provision_demo_infra.yml
   ```

2. **Run the provisioning playbook:**

   ```bash
   ansible-playbook -i /dev/null \
     --private-key ~/.ssh/your-key.pem \
     playbooks/provision_demo_infra.yml \
     -e @vars/demo_infra.yml
   ```

   *(If you encrypted secrets in a vault file, add `--vault-password-file ~/.vault_pass` or `--ask-vault-pass`.)*

3. **Verify the run:**
   * Confirm `aws_demo_infra` created `satellite-sg` and `rhel-sat-nodes-sg` and launched the EC2 instances.
   * Confirm `satellite_server_setup` configured swap, enabled RHSM repos, validated certificates with `katello-certs-check`, and completed `satellite-installer`.
   * Confirm `rhel9-sat01.samboxaws.org` and `rhel10-sat01.samboxaws.org` have their hostnames and `/swapfile` configured.

## 5. Run from AAP 2.7

1. **Sync your Project:** Commit your changes to Git and sync your **Project** in **Automation Execution → Projects**.
2. **Attach Credentials:** On your **Job Template**, attach your **Amazon Web Services** credential and **Machine** (SSH private key) credential.
3. **Enable Prompt on Launch:** Enable **Prompt on launch** for **Variables** on the Job Template.
4. **Launch with Extra Vars:** Provide your environment variables (such as `vpc_id`, `subnet_id`, `key_name`, `rhel9_ami_id`, `rhel10_ami_id`, `satellite_admin_password`, and `letsencrypt_email`) in the launch prompt or a survey/extra vars credential.

## 6. Post-Provisioning Steps

After the playbook finishes:

1. Point your DNS (`A` record) for `satellite.samboxaws.org` to the Satellite instance's public IP if not already configured (required before standalone Let's Encrypt HTTP-01 validation succeeds).
2. Log into Red Hat Satellite (`https://satellite.samboxaws.org`), upload your subscription manifest, sync RHEL 9 and RHEL 10 repositories (`on_demand`), publish your `RHEL-SOE` Content View to the `Dev` lifecycle environment, and create `ak-rhel9-dev` and `ak-rhel10-dev`.
3. Register `rhel9-sat01.samboxaws.org` and `rhel10-sat01.samboxaws.org` to Satellite and Red Hat Lightspeed, then sync your Satellite dynamic inventory source in AAP 2.7.