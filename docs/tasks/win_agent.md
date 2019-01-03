# Task: Cohesity Agent Management - Windows

## Table of Contents
- [Synopsis](#synopsis)
- [Requirements](#requirements)
- [Ansible Variables](#Ansible-Variables)
- [Customizing your playbooks](#Customizing-your-playbooks)
  - [Installation of Cohesity Agent on Windows hosts](#Installation-of-Cohesity-Agent-on-Windows-hosts)
  - [Installation of Cohesity Agent on Windows hosts installed using Service Account](#Installation-of-Cohesity-Agent-on-Windows-hosts-installed-using-Service-Account)
- [How the Task works](#How-the-Task-works)

## SYNOPSIS
[top](#task-cohesity-agent-management-windows)

This task can be used to install and manage the Cohesity Agent on Windows hosts

#### How it works
- The task starts by installing the latest version of the agent will be installed (*state=present*) or removed (*state=absent*) from the `windows` server.
- If *state=present*, then the Windows Firewall rule will be updated to allow the Cohesity Agent to communicate to the Cluster.
- When the *installer_type=volcbt* and *cohesity_agent.reboot=True*, the host will be rebooted to allow the driver to load (*state=present*) or unload (*state=absent*).

### Requirements
[top](#task-cohesity-agent-management-windows)

* Cohesity Cluster running version 6.0 or higher
* Ansible >= 2.6
  * [Ansible Control Machine](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#control-machine-requirements) must be a unix system running any of the following operating systems: Linux (Red Hat, Debian, CentOS), macOS, any of the BSDs. Windows isn’t supported for the control machine.
* Powershell >= 4.0

> **Notes**
  - Currently, the Ansible Module requires Full Cluster Administrator access.
  - Prior to using theis task, refer to the [Setup](/setup.md) and [How to use](/how-to-use.md) sections of this guide.

## Ansible Variables
[top](#task-cohesity-agent-management-windows)

The following is a list of variables and the configuration expected when leveraging this task in your playbook.  For more information on these variables, see the [official Cohesity Ansible module documentation](/modules/cohesity_win_agent.md?id=syntax)
```yaml
cohesity_agent:
  state: "present"
  service_user: "cohesityagent"
  service_password: ""
  install_type: "volcbt"
  preservesettings: False
  reboot: True
```
## Customizing your playbooks
[top](#task-cohesity-agent-management-windows)

This example show how to include the Cohesity-Ansible role in your custom playbooks and leverage this task as part of the delivery.

### Installation of Cohesity Agent on Windows hosts
[top](#task-cohesity-agent-management-windows)

Here is an example playbook that installs the Cohesity agent on all `windows` hosts. Please change it as per your environment.
> **Note:**
  - Prior to using these example playbooks, refer to the [Setup](/setup.md) and [How to use](/how-to-use.md) sections of this guide.

```yaml
---
  - hosts: windows
    # => Please change these variables to connect
    # => to your Cohesity Cluster
    vars:
        var_cohesity_server: cohesity_cluster_vip
        var_cohesity_admin: admin
        var_cohesity_password: admin
        var_validate_certs: False
    gather_facts: no
    roles:
        - cohesity.ansible
    tasks:
      - name: Install new Cohesity Agent on each Windows Physical Server
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
                reboot: True
        tags: [ 'cohesity', 'agent', 'install', 'physical', 'windows' ]
```

### Installation of Cohesity Agent on Windows hosts installed using service account
[top](#task-cohesity-agent-management-windows)

Here is an example playbook that installs the Cohesity agent on all `windows` hosts using a service account and the `filecbt` driver. Please change it as per your environment.
> **Note:**
  - Prior to using these example playbooks, refer to the [Setup](/setup.md) and [How to use](/how-to-use.md) sections of this guide.

```yaml
---
  - hosts: windows
    # => Please change these variables to connect
    # => to your Cohesity Cluster
    vars:
        var_cohesity_server: cohesity_cluster_vip
        var_cohesity_admin: admin
        var_cohesity_password: admin
        var_validate_certs: False
        var_service_user: "cohesity\\svc_account"
        var_service_password: "secret"
    gather_facts: no
    roles:
        - cohesity.ansible
    tasks:
      - name: Install new Cohesity Agent on each Windows Physical Server
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
                service_user: "{{ var_service_user }}"
                service_password: "{{ var_service_password }}"
                install_type: "filecbt"
                reboot: False
        tags: [ 'cohesity', 'agent', 'install', 'physical', 'windows' ]
```

## How the Task works
[top](#task-cohesity-agent-management-windows)

The following information is copied directly from the included task in this role.  The source file can be found at the root of this repo in `/tasks/win_agent.yml`
```yaml
---
- name: "Cohesity agent: Set Agent to state of {{ cohesity_agent.state | default('present') }}"
  cohesity_win_agent:
    cluster: "{{ cohesity_server }}"
    username: "{{ cohesity_admin }}"
    password: "{{ cohesity_password }}"
    validate_certs: "{{ cohesity_validate_certs }}"
    state: "{{ cohesity_agent.state }}"
    service_user: "{{ cohesity_agent.service_user | default('') }}"
    service_password: "{{ cohesity_agent.service_password | default('') }}"
    preservesettings: "{{ cohesity_agent.preservesettings | default(False)}}"
    install_type: "{{ cohesity_agent.install_type | default('volcbt') }}"
  tags: always
  register: installed

- name: Firewall rule to allow CohesityAgent on TCP port 50051
  win_firewall_rule:
    name: Cohesity Agent Ansible
    description:
      - Automated Firewall rule created by the Cohesity Ansible integration to allow
      - for the Cohesity Agent to communicate through the firewall.
    localport: 50051
    action: allow
    direction: in
    protocol: tcp
    state: "{{ cohesity_agent.state }}"
    enabled: yes
  tags: always

# => This reboot will only be triggered if both of the following conditions are true:
# => - The registered variable 'installed' returns True when the changed state is queried.
# => - The user defined variable 'cohesity_win_agent_reboot' returns as True.
- name: Reboot the Hosts after agent modification
  win_reboot:
    reboot_timeout: 180
  when:
    - installed.changed
    - cohesity_agent.reboot
  tags: always
```