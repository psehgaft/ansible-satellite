# ansible-satellite | Satellite 6.12.x Orchestration

### Install and configure Satellite 6.12.x on Red Hat Enterprise Linux 6.x and 7.x. This collection can also be used to set up Satellite on AWS.

Fundamental steps are based on the process outlined at the [Satellite 6.1 Installation Guide on the Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_satellite/6.12/html-single/installing_satellite_server_in_a_connected_network_environment/index).

At the last revision of this document, the current stable version of Satellite is 6.7

Reference [standup.yml](standup.yml), which is the installation playbook, to see an example of how the playbooks may be structured, or take a look at any of the listed playbooks below.

### **ansible-satellite roles:**
_The following roles are called by several playbooks to orchestrate tasks on the Satellite server. Please review the playbooks to see how these come together to work._

1. [satellite-auth](#satellite-authentication-satellite-auth)
1. [satellite-content](#satellite-content-satellite-content)
1. [satellite-install](#satellite-installation-satellite-install)
1. [satellite-maintenance](#satellite-maintenance-tasks-satellite-maintenance)
1. [satellite-route53](#amazon-route53-dns-registration-satellite-route53)
1. [satellite-selfsubscribe](#satellite-self-subscription-satellite-selfsubscribe)
1. [satellite-setup](#satellite-setup-satellite-setup)
1. [satellite-upgrade](#satellite-in-place-upgrade-satellite-upgrade)

### **ansible-satellite playbooks:**
_These playbooks are executed by Ansible Core or Ansible Tower._

1. [ec2_content_hosts_cleanup.yml](#ec2_content_hosts_cleanupyml)
1. [ec2_content_hosts_report.yml](#ec2_content_hosts_reportyml)
1. [customer_portal_api_test.yml](#customer_portal_api_testyml)
1. [maintenance.yml](#maintenanceyml)
1. [refresh_ldap_groups.yml](#refresh_ldap_groupsyml)
1. [refresh_s3_rpms.yml](#refresh_s3_rpmsyml)
1. [self-subscribe.yml](#self-subscribeyml)
1. [standup.yml](#standupyml)
1. [upgrade.yml](#upgradeyml)

### **emergency shell scripts:**
_These scripts are written to aid in refreshing subscriptions on all the hosts, based on the .csv file that is generated by [ec2_content_hosts_report.yml](#ec2_content_hosts_reportyml). These are only for emergencies, when the Satellite server is scheduled to be rebuilt. **These depend on the .csv list of the systems generated by that playbook, so in the case of rebuilding Satellite, run that playbook first to make sure you have that file to reference.** Usage of these scripts assumes that you have access to the SSH keys for your AWS instances, and that they are placed in your ${HOME}/.ssh/ directory._

1. [bash-refresh_subscriptions.sh](#bash-refresh_subscriptionssh)
1. [bash-recreate_subscriptions.sh](#bash-recreate_subscriptionssh)

---

# Roles

## Satellite Authentication (**satellite-auth**)
_This role sets up the Satellite Server with authenticated local users, or ties it into a central LDAP server for authentication._

Invoke the role in the following way. Please note the configuration values specified in [roles/satellite-auth/vars/main.yml](roles/satellite-auth/vars/main.yml), [all.yml](group_vars/all.yml) and [secrets.yml](group_vars/secrets.yml).

```yaml
---
- hosts: satellite6-server-prod
  become: yes
  vars_files:
    - group_vars/all.yml
    - group_vars/secrets.yml
  gather_facts: yes
      # satellite-auth | Define users and assign them roles
    - role: satellite-auth
      # local_users: yes
      # ldap_users: yes
      # ldap_refresh: yes
```

# Ansible Satellite Transition
This playbook will move nodes registered in one Satellite host to another. *It will not install satellite/katello server for you.* Use this after you have set up a new Satellite server and want to move all your existing nodes from one server to another.

# Configure Playbook
Copy the hosts.template file and fill it out with information for your infrastructure. Add systems to [nodes] for hosts you want tasks to run on.

```[satellite]
satellite.example.com

[old_satellite]
satellite.example.com

[puppet_master]
satellite.example.com

[puppet_ca]
satellite.example.com

[nodes]
node1.example.com
node2.example.com
 
[6RedHatEnterpriseServer:vars]
activationkey='server,6epel'

[7RedHatEnterpriseServec:vars]
activationkey='workstation,6epel'
```
Use activation keys to register the hosts so make sure your activation keys are set up in satellite before running.

#Running

Enable the satellite settings create_new_host_when_facts_are_uploaded and create_new_host_when_report_is_uploaded to have hosts automatically created after puppet runs. You should also enable a default_location and default_organization in satellite. These settings are all under the puppet tab.

To run on all of your nodes (defined in hosts) make sure you update the activationkey variables (in hosts) and then use.

`ansible-playbook -i hosts satellite-playbook.yaml`

Add `-k` (ssh) or `-K` (sudo) if you need password prompts.

You can also run just the puppet registration tasks with

`ansible-playbook -i hosts satellite-playbook.yaml --tags puppet`

After the tasks complete you should have new unmanaged hosts in satellite. Edit the host and add any configuration you need (host groups, network, puppet). Unfortunately, I could not find a way to automate those steps yet. Your best bet is probably [hammer](https://github.com/theforeman/hammer-cli).

Once the hosts have been moved you may need to reinstall the katello-agent. Do that with `ansible all -i hosts -m yum -a "state=absent name=katello-agent"` and then `ansible all -i hosts -m yum -a "state=present name=katello-agent"`


## Satellite Content (**satellite-content**)
_This role creates lifecycle environments on the Satellite Server, creates content views and filters them, then sets up activation keys pointing to each, and a release version with wich to activate RHEL systems._

Invoke the role in the following way. Please note the configuration values specified in [roles/satellite-content/vars/main.yml](roles/satellite-content/vars/main.yml), [all.yml](group_vars/all.yml) and [secrets.yml](group_vars/secrets.yml).

```yaml
---
- hosts: satellite6-server-prod
  become: yes
  vars_files:
    - group_vars/all.yml
    - group_vars/secrets.yml
  gather_facts: yes
  roles:
    - role: satellite-content
```

## Satellite Installation (**satellite-install**)
_This role installs Satellite to a RHEL host._

Invoke the role in the following way. Please note the configuration values specified in [roles/satellite-install/vars/main.yml](roles/satellite-install/vars/main.yml), [all.yml](group_vars/all.yml) and [secrets.yml](group_vars/secrets.yml).

```yaml
---
- hosts: satellite6-server-prod
  become: yes
  vars_files:
    - group_vars/all.yml
    - group_vars/secrets.yml
  gather_facts: yes
    # satellite-install | Install Satellite 6 to a host
    - role: satellite-install

```

## Satellite Maintenance Tasks (**satellite-maintenance**)
_This role covers several items with regard to maintaining the security of the Satellite server, such as SSL configuration. It also provides orchestration of rpm content to the Satellite server, so that it can be made available to hosts on a regular basis. It leverages some variables from the **satellite-content** role as well._

Invoke the role in the following way. Please note the configuration values specified in [satellite-maintenance/vars/main.yml](roles/satellite-maintenance/vars/main.yml), [satellite-content/vars/main.yml](roles/satellite-content/vars/main.yml), [all.yml](group_vars/all.yml) and [secrets.yml](group_vars/secrets.yml).

```yaml
---
- hosts: satellite6-server-prod
  become: yes
  gather_facts: yes
  roles:
    # satellite-maintenance | Apply maintenance or tweaks
    - role: satellite-maintenance
      security_tweaks: yes
      # upload_rpms: no
      # autoupdate_content_views: no
      # promote_content_views_to_prod: no
```

## Amazon Route53 DNS Registration (**satellite-route53**)
_This role adds an entry into Amazon Route53 DNS for the Satellite server._

Invoke the role in the following way. Please note the configuration values specified in  [all.yml](group_vars/all.yml).

```yaml
---
- hosts: satellite6-server-prod
  become: yes
  vars_files:
    - group_vars/all.yml
    - group_vars/secrets.yml
  gather_facts: yes
  roles:
    - role: satellite-route53
      # Won't register in DNS if you don't set this to true
      register_route53: True
```

## Satellite Self-Subscription (**satellite-selfsubscribe**)
_This roles subscribes the Satellite server to itself. It pauses for a period to allow someone to update the Satellite server manifest at the **Red Hat Customer Portal > Subscription Management > [Subscription Management Applications](https://access.redhat.com/management/distributors?type=satellite) > Satellite**, and will then continue to set Satellite up to receive content filtered in the same way as other systems._

Invoke the role in the following way. Please note the configuration values specified in [roles/satellite-selfsubscribe/vars/main.yml](roles/satellite-selfsubscribe/vars/main.yml),  [all.yml](group_vars/all.yml) and [secrets.yml](group_vars/secrets.yml).

```yaml
---
- hosts: satellite6-server-prod
  become: yes
  vars_files:
    - group_vars/all.yml
    - group_vars/secrets.yml
  gather_facts: yes
  roles:
    # satellite-selfsubscribe | Subscribe the server to itself
    - role: satellite-selfsubscribe
```

---

#### Prerequisite to using the **satellite-setup** role, you must create a manifest at the **Red Hat Customer Portal > Subscription Management > [Subscription Management Applications](https://access.redhat.com/management/distributors?type=satellite) > Satellite** and add it to the role in the files folder.

Note: A manifest can been created and included as part of this playbook. It can be overwritten, and/or refreshed from Satellite after it has been imported. It will then pull the up-to-date subscription information from Red Hat. The manifest should be in [roles/satellite-setup/files/manifest.zip](roles/satellite-setup/files/manifest.zip).

---

## Satellite Setup (**satellite-setup**)
_This role ties the Satellite server to Red Hat using the manifest mentioned above, activates products, repositories, and also brings in Docker images from the Red Hat Registry, along with 3rd party and custom repositories for your own generated RPM content._

Invoke the role in the following way. Please note the configuration values specified in [roles/satellite-setup/vars/main.yml](roles/satellite-setup/vars/main.yml), [all.yml](group_vars/all.yml) and [secrets.yml](group_vars/secrets.yml)

```yaml
---
- hosts: satellite6-server-prod
  become: yes
  vars_files:
    - group_vars/all.yml
    - group_vars/secrets.yml
  gather_facts: yes
  roles:
    - role: satellite-setup
```

## Satellite In-Place Upgrade (**satellite-upgrade**)
_This role performs an in-place upgrade of Satellite 6.1 to the current 6.1.x release._

Invoke the role in the following way. Please note the configuration values specified in  [all.yml](group_vars/all.yml).

```yaml
---
- hosts: satellite6-server-prod
  become: yes
  gather_facts: yes
  roles:
    # satellite-upgrade | Perform Satellite Upgrade
    - role: satellite-upgrade
      slack_upgrade_notify: yes
```

---

# Playbooks

## [ec2_content_hosts_cleanup.yml](ec2_content_hosts_cleanup.yml)
_Ansible Tower scheduled job, that removes systems that are registered in Satellite if they are not present in EC2 inventory._

**Inline tasks. No roles invoked.**

## [ec2_content_hosts_report.yml](ec2_content_hosts_report.yml)
_Queries the Satellite API and AWS CLI, then generates a .csv file with details about hosts that are subscribed to Satellite, pulling in security key and other information from EC2. This is needed to resubscribe servers in a scripted fashion._

**Inline tasks. No roles invoked.**

## [customer_portal_api_test.yml](customer_portal_api_test.yml)
_Starts to test some pulling of entitlement information from the Red Hat Customer Portal, via the Candlepin API._

**Inline tasks. No roles invoked.**

## [maintenance.yml](maintenance.yml)
_Called ad-hoc to perform tasks, post-installation. Can update the custom RPMs uploaded from S3, update the content views with the most current packages, promote those to production, and also apply OpenSSL security tweaks to Apache._

| Roles Invoked | Extra Vars |
| --- | --- |
| **satellite-maintenance** | `upload_rpms` (boolean), `autoupdate_content_views` (boolean), `promote_content_views_to_prod` (boolean), `security_tweaks` (boolean), `slack_pubpromo_notify` (boolean),  `restart_services` (boolean) |

## [refresh_ldap_groups.yml](refresh_ldap_groups.yml)
_Ansible Tower scheduled job, that refreshes the groups internal to Satellite with information from their counterparts in Active Directory, then assigns roles and users as needed._

| Roles Invoked | Extra Vars |
| --- | --- |
| **satellite-auth** | `ldap_refresh` (boolean) |

## [refresh_s3_rpms.yml](refresh_s3_rpms.yml)
_Ansible Tower scheduled job, that downloads RPMs from the bucket specified in the configuration variables, then uploads them to custom repositories in Satellite if they have changed._

| Roles Invoked | Extra Vars |
| --- | --- |
| **satellite-maintenance** | `upload_rpms` (boolean) |

## [self-subscribe.yml](self-subscribe.yml)
_Subscribes the Satellite server to itself, to take advantage of the ability to release updates on a cycle, with the rest of the systems. Requires some manual interation with the Red Hat Customer Portal. Afterwards, runs the ansible-common playbook against Satellite._

| Roles Invoked | Extra Vars |
| --- | --- |
| **satellite-selfsubscribe** | n/a |
| **satellite-common (ansible-common)** | `yum_update` (boolean) |

## [standup.yml](standup.yml)
_Spins up an AWS instance, installs Satellite, brings in entitlements and content, and then makes it available for consumption by systems._

| Roles Invoked | Extra Vars |
| --- | --- |
| **satellite-install** | n/a |
| **satellite-setup** | n/a |
| **satellite-route53** | n/a |
| **satellite-auth** | `local_users` (boolean), `ldap_users` (boolean) |
| **satellite-content** | n/a |
| **satellite-maintenance** | `security_tweaks` (boolean) |

## [upgrade.yml](upgrade.yml)
_Performs an in-place upgrade of the Satellite server._

| Roles Invoked | Extra Vars |
| --- | --- |
| **satellite-upgrade** | `slack_upgrade_notify` (boolean) |
| **satellite-maintenance** | `security_tweaks` (boolean) |

---

# Emergency Shell Scripts

## [bash-refresh_subscriptions.sh](bash-refresh_subscriptions.sh)
_Refresh subscriptions on all content hosts, iterating on an exported .csv file_

Usage: # `./bash-refresh_subscriptions.sh ${pathToContentHostCSV}`

## [bash-recreate_subscriptions.sh](bash-recreate_subscriptions.sh)
_Unsubscribe and resubscribe all content hosts, iterating on an exported .csv file_

Usage: # `./bash-recreate_subscriptions.sh ${pathToContentHostCSV} ${satellite_fqdn} ${organization_name}`
