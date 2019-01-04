# Cohesity Agent uninstallation and installation using Ansible Inventory

## SYNOPSIS
This example play employs the Ansible Inventory to dynamically remove and install the Cohesity Agent on the supported platforms.
- The play starts by reading all `linux` servers from the Ansible Inventory and uninstalling the agent (if present).
- Upon completion of agent removal, the latest version of the agent is installed on the `linux` servers.
- The next step is to perform the uninstallation of the Agent on `windows` servers.
  - This is followed by a mandatory reboot of the server as a part of the uninstallation (only if the agent was present).
- Once the reboot is complete, the latest version of the agent is installed on the `windows` servers.
  - If the windows `install_type` is `volcbt`, then a reboot of the windows servers is triggered.

**NOTE**: This example play is included only as a reference. This will connect to all `linux` and `windows` servers and remove the previous agent, then install the latest Cohesity agent. There are no job validations or state checks to ensure that backups are not running.

> **Tip:**  Currently, the Cohesity Ansible Role requires Cohesity Cluster Administrator access.

## Ansible Variables

| Required | Parameters | Type | Choices/Defaults | Comments |
| --- | --- | --- | --- | --- |
| X | **var_cohesity_server** | String | | IP or FQDN for the Cohesity cluster |
| X | **var_cohesity_admin** | String | | Username with which Ansible will connect to the Cohesity cluster |
| X | **var_cohesity_password** | String | | Password belonging to the selected Username.  This parameter is not logged. |
|   | var_validate_certs | Boolean | False | Switch that determines whether SSL Validation is enabled. |

## Example Playbook

This is an example playbook that deploys the Cohesity agent on Linux and Windows. Before using it, edit it to address your own environment.

```yaml
# => Cohesity Agent Management
# =>
# => Role: cohesity.ansible
# =>

# => Install the Cohesity Agent on each Linux and Windows host
# => specified in the Ansible inventory
# =>
---
  - hosts: linux
    # => We need to specify these variables to connect
    # => to the Cohesity cluster
    vars:
        var_cohesity_server: cohesity_cluster_vip
        var_cohesity_admin: admin
        var_cohesity_password: admin
        var_validate_certs: False
    # => We need to gather facts to determine the OS type of
    # => the machine
    gather_facts: yes
    become: true
    roles:
        - cohesity.ansible
    tasks:
      - name: Uninstall Cohesity Agent from each Linux Server
        include_role:
            name: cohesity.ansible
            tasks_from: agent
        vars:
            cohesity_server: "{{ var_cohesity_server }}"
            cohesity_admin: "{{ var_cohesity_admin }}"
            cohesity_password: "{{ var_cohesity_password }}"
            cohesity_validate_certs: "{{ var_validate_certs }}"
            cohesity_agent:
                state: absent

      - name: Install new Cohesity Agent on each Linux Server
        include_role:
            name: cohesity.ansible
            tasks_from: agent
        vars:
            cohesity_server: "{{ var_cohesity_server }}"
            cohesity_admin: "{{ var_cohesity_admin }}"
            cohesity_password: "{{ var_cohesity_password }}"
            cohesity_validate_certs: "{{ var_validate_certs }}"
            cohesity_agent:
                state: present
  - hosts: windows
    # => We need to specify these variables to connect
    # => to the Cohesity cluster
    vars:
        var_cohesity_server: cohesity_cluster_vip
        var_cohesity_admin: admin
        var_cohesity_password: admin
        var_validate_certs: False
    gather_facts: no
    roles:
        - cohesity.ansible
    tasks:
      - name: Remove Cohesity Agent from each Windows Server
        include_role:
            name: cohesity.ansible
            tasks_from: win_agent
        vars:
            cohesity_server: "{{ var_cohesity_server }}"
            cohesity_admin: "{{ var_cohesity_admin }}"
            cohesity_password: "{{ var_cohesity_password }}"
            cohesity_validate_certs: "{{ var_validate_certs }}"
            cohesity_agent:
                state: absent
                reboot: True

      - name: Install new Cohesity Agent on each Windows Server
        include_role:
            name: cohesity.ansible
            tasks_from: win_agent
        vars:
            cohesity_server: "{{ var_cohesity_server }}"
            cohesity_admin: "{{ var_cohesity_admin }}"
            cohesity_password: "{{ var_cohesity_password }}"
            cohesity_validate_certs: "{{ var_validate_certs }}"
            cohesity_agent:
                state: present
                install_type: volcbt
                reboot: True
```

## Ansible Inventory Configuration

To take full advantage of this Ansible Play, configure your Ansible Inventory File with the keys and values shown. This makes it much easier to manage the overall experience.

Here is an example Inventory File. Before using it, edit it to address your own environment.
```ini
[linux]
10.2.46.96
10.2.46.97
10.2.46.98
10.2.46.99

[linux:vars]
ansible_user=root

[windows]
10.2.45.88
10.2.45.89

[windows:vars]
ansible_user=administrator
ansible_password=secret
ansible_connection=winrm
ansible_winrm_server_cert_validation=ignore
```
