# AWS Demo VM Lifecycle Management

This playbook (`manage_demo_vms.yml`) automates starting and stopping EC2 instances for the Red Hat Lightspeed & Ansible Automation Platform (AAP) demo environment. It targets instances matching the `Name` tag filter (by default `*samboxaws.org*`) in the configured AWS region (`us-east-2`).

---

## 1. Prerequisites

Ensure your local workstation has the required Python packages and Ansible collection installed:

```bash
pip install boto3 botocore
ansible-galaxy collection install amazon.aws
```

---

## 2. Credentials Configuration

The playbook expects an encrypted credentials file named `aws_demo_creds.yml` in the same directory.

### Create/Edit Credentials
Create `aws_demo_creds.yml` with your AWS credentials:

```yaml
---
aws_access_key: "YOUR_AWS_ACCESS_KEY"
aws_secret_key: "YOUR_AWS_SECRET_KEY"
# aws_session_token: "YOUR_OPTIONAL_SESSION_TOKEN"  # Required only for temporary STS or SSO keys
```

### Encrypt with Ansible Vault
Encrypt the file so secrets are not saved in plain text:

```bash
ansible-vault encrypt aws_demo_creds.yml
```

*(To edit the file later, run `ansible-vault edit aws_demo_creds.yml`)*

---

## 3. Playbook Execution

### Stop All Demo VMs (Weekend / Cost Savings)
To gracefully stop all matching EC2 instances:

```bash
ansible-playbook manage_demo_vms.yml --ask-vault-pass -e vm_state=stopped
```

### Start All Demo VMs
To bring all matching EC2 instances back online:

```bash
ansible-playbook manage_demo_vms.yml --ask-vault-pass -e vm_state=running
```

---

## 4. Parameter Overrides

You can pass extra variables (`-e`) to override default settings on the fly:

| Parameter | Default Value | Description | Example Usage |
| :--- | :--- | :--- | :--- |
| `vm_state` | `stopped` | Target state (`running` or `stopped`) | `-e vm_state=running` |
| `aws_region` | `us-east-2` | AWS region where instances reside | `-e aws_region=us-east-1` |
| `name_filter` | `*samboxaws.org*` | EC2 `tag:Name` search pattern | `-e name_filter="*rhel9*"` |

### Example: Target Specific Nodes in a Different Region
```bash
ansible-playbook manage_demo_vms.yml --ask-vault-pass \
  -e vm_state=running \
  -e aws_region=us-east-1 \
  -e name_filter="*rhel9*"
```

---

## 5. Dry-Run Verification (Check Mode)

To test your credentials and see which instances will be targeted without changing their state, run with `--check`:

```bash
ansible-playbook manage_demo_vms.yml --ask-vault-pass --check
```

---

## 6. Important Notes
* **Graceful Shutdown:** EC2 stop actions trigger standard OS shutdown hooks. `systemd` cleanly stops rootless AAP container services (`automation-*`, `redis`, `postgresql`) on the host prior to powering down.
* **IP Addresses:** Target nodes use static private IPs (`ansible_host`). If your AAP controller does not use an AWS Elastic IP, AWS will assign a new public IP upon startup—check the task stdout summary for the updated address.