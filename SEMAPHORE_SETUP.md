# Semaphore Setup Instructions

This playbook is configured to work with Ansible Semaphore and its Key Store.

## Semaphore Key Store Configuration

### 1. Add SSH Keys to Semaphore Key Store

In Semaphore UI:
1. Go to **Key Store**
2. Add two new keys:
   - **Name:** `id_lcs` (for user lcs)
     - **Type:** SSH Key
     - Upload or paste the SSH public and private key pair
   - **Name:** `id_deploy` (for user deploy)
     - **Type:** SSH Key
     - Upload or paste the SSH public and private key pair

### 2. Create Environment Variables

In your Semaphore Project, create **Environment** variables:

1. Go to **Environment** in your project
2. Add these variables:
   ```
   lcs_ssh_key: <paste the public key content for lcs user>
   deploy_ssh_key: <paste the public key content for deploy user>
   ```
   
   OR create a **Vault** file with:
   ```yaml
   ---
   lcs_ssh_key: |
     ssh-rsa AAAAB3NzaC1yc2EA... (your lcs public key)
   deploy_ssh_key: |
     ssh-rsa AAAAB3NzaC1yc2EA... (your deploy public key)
   ```

### 3. Create Inventory

In Semaphore:
1. Go to **Inventory**
2. Create a new inventory with the content from `inventory.yml`
3. Set the **SSH Key** to the root access key for initial connection

### 4. Create Templates

#### Template 1: Deploy Users

- **Name:** Deploy Users - hal.local
- **Playbook:** `1_deploy_users.yml`
- **Inventory:** (select the inventory you created)
- **Repository:** (your git repository)
- **Environment:** (select the environment with SSH keys)
- **Extra Variables:**
  ```yaml
  skip_ssh_test: true
  ```
  *(Note: SSH test is skipped in Semaphore since private keys for testing aren't available)*

#### Template 2: Harden SSH

- **Name:** Harden SSH - hal.local
- **Playbook:** `2_harden_ssh.yml`
- **Inventory:** (same as above)
- **Repository:** (your git repository)
- **Environment:** (same as above)
- **Extra Variables:**
  ```yaml
  require_confirmation: false
  ```
  *(Note: Confirmation is disabled in automated Semaphore runs)*

### 5. Run the Templates

1. **First:** Run "Deploy Users" template
   - This creates users and deploys their SSH keys
   - Verify manually that you can SSH with keys before proceeding

2. **Second:** Run "Harden SSH" template (only after verifying SSH key access!)
   - This disables password authentication
   - After this, only key-based login will work

## Manual Verification Between Steps

After running the "Deploy Users" template, manually test SSH access:

```bash
ssh -i /path/to/id_lcs lcs@hal.local
ssh -i /path/to/id_deploy deploy@hal.local
```

Only proceed with "Harden SSH" if both connections work!

## Important Notes

- **SSH Key Testing:** The automatic SSH key test in the playbook is disabled when `skip_ssh_test: true` because Semaphore doesn't have access to test with private keys during playbook execution
- **Manual Verification Required:** You MUST manually verify SSH key login works before running the hardening playbook
- **Backup Access:** Ensure you have console/IPMI access to the server in case of lockout
- **Root Access:** The initial connection uses root access, which is then disabled by the hardening playbook

## Troubleshooting

If you get locked out:
1. Use console/IPMI access
2. Edit `/etc/ssh/sshd_config`
3. Set `PasswordAuthentication yes`
4. Restart SSH: `systemctl restart sshd`
5. Re-run the Deploy Users template to fix any issues
