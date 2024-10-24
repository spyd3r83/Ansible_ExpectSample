# Dependency

Ensure the following software is installed on your control node (the machine from which you run the playbook):

- **Ansible** (version 2.9 or later)
- **Expect**

# Directory Structure

```plaintext
playbook_directory/
├── change_password.exp
├── change_password.yml
├── host_vars/
│   ├── 10.1.101.102.yml
│   ├── 10.1.101.103.yml
│   └── 10.1.101.104.yml
└── hosts.ini
```

# Expect Script: `change_password.exp`

This Expect script automates the password change process.

```bash
#!/usr/bin/expect -f

# This Expect script automates the password change process

set timeout 120
exp_internal 1
log_file /tmp/expect_debug.log

# Set terminal type to dumb to prevent ANSI codes (optional)
set env(TERM) "dumb"

# Retrieve arguments
set host [lindex $argv 0]
set user [lindex $argv 1]
set initial_password [lindex $argv 2]
set new_password [lindex $argv 3]

# Start SSH session
spawn ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null $user@$host

# Handle initial password prompt
expect {
    -re {.[Pp]assword:.} {
        send "$initial_password\r"
    }
    timeout {
        puts "Error: SSH connection timed out."
        exit 1
    }
}

# Handle any motd or login banners until we get to the Shape prompt
expect {
    -re {(\x1B[.*?m)Shape.-> } {
        send "run bash\r"
    }
    timeout {
        puts "Error: Did not receive Shape CLI prompt."
        exit 1
    }
}

# Handle the password prompt after 'run bash'
expect {
    -re {Please input the password for user '.*'} {
        send "$initial_password\r"
    }
    timeout {
        puts "Error: Did not receive password prompt after 'run bash'."
        exit 1
    }
}

# Wait for bash shell prompt
expect {
    -re {[#$] } {
        # First attempt to change password
        send "passwd\r"
    }
    timeout {
        puts "Error: Did not reach bash shell prompt."
        exit 1
    }
}

# Handle password change process
expect {
    -re {(current )?UNIX password:} {
        send "$initial_password\r"
        exp_continue
    }
    "passwd: Authentication token manipulation error" {
        # Retry changing password if first attempt failed
        send "passwd\r"
        exp_continue
    }
    "New password:" {
        send "$new_password\r"
        exp_continue
    }
    "Retype new password:" {
        send "$new_password\r"
        exp_continue
    }
    "BAD PASSWORD.*" {
        send "$new_password\r"
        exp_continue
    }
    "passwd: all authentication tokens updated successfully." {
        # Password change successful
        send "exit\r"
    }
    timeout {
        puts "Error: Password change process timed out."
        exit 1
    }
}

# Exit bash shell
expect {
    -re {[#$] } {
        send "exit\r"
    }
    timeout {
        puts "Error: Did not return to bash shell prompt after password change."
        exit 1
    }
}

# Close SSH session
expect eof
exit 0
```

# Ansible Playbook: `change_password.yml`

This Ansible playbook changes the password on all specified hosts using the Expect script.

```yaml
- hosts: all
  gather_facts: no
  vars:
    ansible_user: admin  # Replace with your actual username if different
  tasks:
    - name: Change password via Expect script
      delegate_to: localhost
      connection: local
      vars:
        host: "{{ inventory_hostname }}"
        user: "{{ ansible_user }}"
        initial_pass: "{{ initial_password }}"
        new_pass: "{{ new_password }}"
      shell: "./change_password.exp {{ host }} {{ user }} '{{ initial_pass }}' '{{ new_pass }}'"
      register: change_password_result
      failed_when: change_password_result.rc != 0
```

# Inventory File: `hosts.ini`

Specify the target hosts in this inventory file.

```ini
10.1.101.102
```

# Host Variables: `host_vars/host.yml` (Vault this File)

Store sensitive passwords in this file and ensure it's vaulted for security.

```yaml
initial_password: 'AllMyMaster123'
new_password: 'NewPassword456'
```

# Example Usage

Execute the following command:

```bash
ansible-playbook -i hosts.ini change_password.yml --ask-vault-pass
```

## Output Example

```bash
ansible-playbook -i hosts.ini change_password.yml --ask-vault-pass
Vault password:

PLAY [all] *************************************************************************************************

TASK [Change password via Expect script] *******************************************************************
changed: [10.1.101.102 -> localhost]

PLAY RECAP *************************************************************************************************
10.1.101.102               : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
