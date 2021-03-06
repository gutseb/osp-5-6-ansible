---
title: RHEL-OSP 5 to RHEL-OSP 6 HA Upgrade
---

# RHEL-OSP 5 to RHEL-OSP 6 HA Upgrade

This is an [Ansible][] playbook that will upgrade a RHEL-OSP 5 HA environment to
RHEL-OSP 6.  Your OpenStack environment will be unavailable for the duration
of the upgrade.

[ansible]: http://www.ansible.com/

In order to use this playbook, you must create a "hosts" file
listing your controllers and compute hosts in the following format:

    [controllers]
    192.168.100.10
    192.168.100.11
    192.168.100.12

    [compute]
    192.168.100.13

And then run ansible like this:

    ansible-playbook -i hosts osp-5-6-ha-upgrade.yaml

You can run specific sections of the playbook through the use of
Ansible's "tags" feature.  For example, to run just the pre-check
plays:

    ansible-playbook -i hosts osp-5-6-ha-upgrade.yaml -t pre-check

You can also *skip* specific sections of the playbook using the
`--skip-tags` option.  For example, to *skip* the pre-check:

    ansible-playbook -i hosts osp-5-6-ha-upgrade.yaml --skip-tags pre-check

You can skip most the validation steps with `--skip-tags verify`.

## A note about this documentation

This documentation was generated automatically from the Ansible
playbook.  You can run `make` at any time to regenerate the
documentation.

## Before you begin

You must update your system configuration to point at RHEL-OSP 6 package
repositories, using either subscription-manager or some other appropriate
tool.


<!-- break -->

    
## Pre-check

Before starting the upgrade process, we check to make sure the current
environment is healthy.  Specifically, we check:

- Is pacemaker alive and responding?
- Are there any inactive resources?
- Is MySQL/MariaDB operational?
- Is Keystone operational?


<!-- break -->

    - hosts: controllers[0]
      tags:
        - pre-check
      tasks:
        # This verifies that pacemaker is responding
        - name: verify that pacemaker is operational
          command: pcs status
        # This checks that all top-level resources are active, and that all
        # resource containers (resource groups, clones, etc) have at least one
        # active resource.
        - name: checking for inactive resources
          shell: |
            tmpfile=$(mktemp xmlXXXXXX)
            trap "rm -f $tmpfile" EXIT
    
            pcs status xml > $tmpfile
            xmllint --xpath '/crm_mon/resources/*/@id' $tmpfile | tr ' ' '\n'| cut -f2 -d'"' |
            while read id; do
              [ "$id" ] || continue
              if ! xmllint --xpath "/crm_mon/resources/*[@id='$id' and @active='true']|/crm_mon/resources/*[@id='$id']/*[@active='true']" $tmpfile > /dev/null 2>&1; then
                echo "$id: no active resources" >&2
                exit 1
              fi
            done
        # This ensures that the database is running and that we have
        # appropriate credentials available.
        - name: verify that database is operational
          command: mysql -e "select 1"
        # This verifies that keystone is up and running
        - name: verify that keystone is operational
          shell: |
            . /root/keystonerc_admin
            keystone token-get
    
## Basic configuration tasks

In order to prevent conflicts during (and after) the upgrade process, we need
to disable Puppet.  We also put selinux into permissive mode to work around
existing bugs in the RHEL-OSP 6 packages for RHEL 7.0.


<!-- break -->

    - hosts: all
      tasks:
        # This is only necessary because there are outstanding selinux bugs in
        # RHEL-OSP 6 for RHEL 7.0.  For details, see
        #
        #   https://bugzilla.redhat.com/show_bug.cgi?id=1185444
        #
        # You can disable this task with `--skip-tags bz1185444`.
        - name: make selinux permissive
          tag:
            - bugfix
            - bz1185444
          command: setenforce 0
    
        # We need to disable Puppet to prevent it from reverting changes to
        # configuration files made during this upgrade process.
        - name: disable puppet
          service: name=puppet enabled=false state=stopped
          ignore_errors: true
    
    - hosts: controllers
      tasks:
        # In RHEL-OSP 5, some services were erroneously configured to be started by
        # systemd *and* by Pacemaker.  We want services involved in our HA
        # configuration to be managed only by Pacemaker.
        - name: prevent systemd from managing some services
          service: name={{item}} enabled=false
          with_items:
            - memcached
            - haproxy
    
## Stop all resources

We instruct Pacemaker to stop all managed resources, and then wait for
everything to stop completely before continuing.  This avoids conflicts as we
update packages and modify configuration files and resource configurations
during the upgrade process.

After Pacemaker stops all the resources, we mark them individually disabled and then 
unset Pacemaker's `stop-all-resources` property.  This allows Pacemaker to
start resources as we create them or explicitly enable them as part of the
upgrade.


<!-- break -->

    - hosts: controllers[0]
      tasks:
        - name: stop all pacemaker managed resources
          command: pcs property set stop-all-resources=true
        - name: wait for all resources to stop
          shell: |
            while pcs status xml |
                xmllint --xpath '//resource[@active="true"]' -; do
              sleep 1
            done
    
        # Explicitly disable all resources, so that when we set
        # stop_all_resources=false Pacemaker will not start anything.
        - name: disable all resources
          shell: |
            cibadmin -Q |
              xmllint --xpath '/cib/configuration/resources/*/@id' - |
              tr ' ' '\n' |
              cut -f2 -d'"' |
              xargs -n1 pcs resource disable
    
        # This will not start anything immediately, but it means that as
        # we create or enable resources later on in this playbook,
        # Pacemaker will start the resources.
        - name: allow pacemaker to start resources again
          command: pcs property set stop-all-resources=false
    
## Stop services on compute hosts

The compute hosts are not part of the Pacemaker cluster, so we need
to shut down services there separately.


<!-- break -->

    - hosts: compute
      tags:
        - compute
      tasks:
        - name: stop all openstack services
          command: openstack-service stop
    
## Upgrade all packages

We run `yum upgrade` on all the hosts in our OpenStack deployment to
install the RHEL-OSP 6 packages.


<!-- break -->

    - hosts: all
      tasks:
        - name: upgrade all packages
          command: yum -y upgrade
    
## Reconfigure VIP resources

RHEL-OSP 6 names VIP resources differently from RHEL-OSP 5.  In this step, we
remove all the existing VIP resources and then re-generate them, using
RHEL-OSP 6 naming conventions, from information in the HAProxy configuration.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - haproxy
      tasks:
        - name: delete old vip resources
          shell: |
            crm_resource -l |
              grep ip- |
              xargs -n1 pcs resource delete
    
        # This rather hairy chunk of code handles the renaming of VIP
        # resources between RHEL-OSP 5 and RHEL-OSP 6.
        - name: create new vip resources
          shell: |
            egrep 'listen|bind' /etc/haproxy/haproxy.cfg |
            { while read entry serv; do
              if [ "$entry" = listen ]; then
                #echo $entry
                case "$serv" in
                  amqp)             vipserv=amqp;;
                  cinder-api)       vipserv=cinder;;
                  galera)           vipserv=galera;;
                  glance-api)       vipserv=glance;;
                  glance-registry)  vipserv="";;
                  heat-api)         vipserv=heat;;
                  heat-cfn)         vipserv=heat_cfn;;
                  heat-cloudwatch)  vipserv="";;
                  horizon)          vipserv=horizon;;
                  keystone-admin)   vipserv=keystone;;
                  keystone-public)  vipserv="";;
                  neutron-api)      vipserv=neutron;;
                  nova-api)         vipserv=nova;;
                  nova-metadata)    vipserv="";;
                  nova-novncproxy)  vipserv="";;
                  nova-xvpvncproxy) vipserv="";;
                  stats)            vipserv="";;
    
                  *) echo UNKNOWN && exit 1;;
                esac
                suffix=pub
              fi
    
              if [ -n "$vipserv" ] && [ "$entry" = bind ]; then
                # note this only work for IPv4
                serv=${serv%:*}
                pcs resource create ip-$vipserv-$suffix-$serv \
                  IPaddr2 ip=$serv cidr_netmask=32
                pcs constraint order start ip-$vipserv-$suffix-$serv \
                  then haproxy-clone kind=Optional
                pcs constraint colocation add ip-$vipserv-$suffix-$serv \
                  with haproxy-clone
                # add pub with prv with adm constraints!
                [ "$suffix" != pub ] && pcs constraint colocation add \
                  ip-$vipserv-$suffix-$serv with ip-$vipserv-pub-$pubip
                [ "$suffix" = prv ] && suffix=adm
                if [ "$suffix" = pub ]; then
                  pubip=$serv
                  suffix=prv
                fi
              fi
            done; }
        # XXX: why do notes have no start-delay setting for haproxy?
        - name: update haproxy resource configuration
          shell: |
            pcs resource update haproxy start-delay=
            pcs resource update haproxy op monitor interval=60s
            pcs resource update haproxy clone interleave=true
        - name: enable haproxy resource
          command: pcs resource enable haproxy-clone
    
## Update MySQL/MariaDB resource

RHEL-OSP 6 uses a different resource agent for managing the database.  The `galera`
agents prevents issues encountered in the older configuration that would
prevent the database from starting after rebooting all the nodes in the
Pacemaker cluster.

In this step, we remove parts of the configuration file specific to the older
configuration.


<!-- break -->

    - hosts: controllers
      tags:
        - database
      tasks:
        # Remove the legacy replication configuration from the
        # MySQL/MariaDB configuration files.
        - name: edit galera configuration file
          command: >
            sed -i 's/^wsrep_cluster_address/#wsrep_cluster_address/g' 
              /etc/my.cnf.d/galera.cnf
    
Now that the configuration files are correct, we delete the old
`mysqld` resource and create a new `galera` resource.  Pacemaker
will proceed to start this resource, and we wait for the database to
become operational before continuing.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - database
      tasks:
        # In order to make this playbook idempotent, we attempt to delete
        # both the old "mysqld" and the new "galera" resource (and then
        # subsequently create the "galera" resource).
        - name: check if mysqld resource exists
          command: crm_resource -q -r {{item}}
          ignore_errors: true
          register: database
          with_items:
            - mysqld
            - galera
        - name: delete mysqld resource
          command: pcs resource delete {{item.item}}
          when: item.rc == 0
          with_items: database.results
        - name: create new mysqld resource using galera resource agent
          shell: |
            nodes=$(pcs status | grep ^Online | sed -e 's/.*\[ //g' -e 's/ \].*//g' -e 's/ /,/g')
            pcs resource create galera galera enable_creation=true \
              wsrep_cluster_address="gcomm://$nodes" meta master-max={{groups.controllers|length}} \
              ordered=true op promote timeout=300s on-fail=block  --master
        - name: wait for database to become active
          tags:
            - verify
          shell: |
            timeout 300 sh -c 'until mysql -e "select 1"; do sleep 1; done'
    
## Update rabbitmq resource

In this step we replace the RHEL-OSP 5 `rabbitmq-server` resource
with an identically named resource using the new `rabbitmq-cluster`
resource agent.

We start by removing parts of the configuration specific to the
older resource definition.


<!-- break -->

    - hosts: controllers
      tags:
        - rabbitmq
      tasks:
        # Remove parts of the rabbitmq configuration used in RHEL-OSP 5 HA
        # configuration.
        - name: prepare rabbitmq configuration
          shell: |
            sed -i '/cluster_/d' /etc/rabbitmq/rabbitmq.config 
            rm -rf /var/lib/rabbitmq/mnesia/
    
Then we replace the `rabbitmq-server` resource definition, enable
the resource, and wait for the service to become active.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - rabbitmq
      tasks:
        - name: delete old rabbitmq resource
          command: pcs resource delete rabbitmq-server
        - name: create new rabbitmq resource using rabbitmq-cluster agent
          command: >
            pcs resource create rabbitmq-server rabbitmq-cluster
              set_policy='HA ^(?!amq\.).* {"ha-mode":"all"}'
              clone ordered=true
        - name: wait for rabbitmq to become active
          tags:
            - verify
          shell: |
            timeout 300 sh -c 'until rabbitmqctl status; do sleep 1; done'
    
## Update memcached resource

This makes some minor changes to the `memcached` resource and then
enables the service.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - memcached
      tasks:
        - name: update memcached resource
          shell: |
            pcs resource update memcached start-delay=
            pcs resource update memcached op monitor interval=60s
            pcs resource update memcached clone interleave=true
        - name: enable memcached resource
          command: pcs resource enable memcached-clone
    
## Update OpenStack database schemas

We use the `openstack-db` wrapper script to perform database schema
upgrades on all of our OpenStack services.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - database
      tasks:
        - name: update openstack database schemas
          command: openstack-db --service {{item}} --update
          with_items:
            - keystone
            - glance
            - cinder
            - nova
            - neutron
            - heat
    
## Update Keystone resource

In this step we make some minor changes to the `openstack-keystone`
resource, and we add some missing start constraints such that
keystone will start *after* basic services (galera, rabbitmq,
memcached, haproxy) are up.

We enable the resource and wait for keystone to become active before
continuing.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - keystone
      tasks:
        - name: add startup order constraints on keystone
          command: >
            pcs constraint order start {{item}}
              then openstack-keystone-clone
          with_items:
            - galera-master
            - rabbitmq-server-clone
            - memcached-clone
            - haproxy-clone
        - name: update keystone resource
          shell: |
            pcs resource update openstack-keystone start-delay=
            pcs resource update openstack-keystone op monitor interval=60s
            pcs resource update openstack-keystone clone interleave=true
        - name: enable keystone resource
          command: pcs resource enable openstack-keystone-clone
        - name: wait for keystone to become active
          tags:
            - verify
          shell: |
            . /root/keystonerc_admin
            timeout 300 sh -c 'until keystone token-get; do sleep 1; done'
    
## Update Glance resources

This steps updates both the `openstack-glance-api` and
`openstack-glance-registry` resources.  We add a start constraint on
glance-registry such that it will only start *after* keystone (there is
an existing constraint between glance-api and glance-registry that
ensures the proper ordering of those services).

Then we enable glance (and the supporting filesystem), and wait for
glance to become active before continuing.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - glance
      tasks:
        - name: update glance resources
          shell: |
            pcs resource update {{item}} start-delay=
            pcs resource update {{item}} op monitor interval=60s
            pcs resource update {{item}} clone interleave=true
          with_items:
            - openstack-glance-api
            - openstack-glance-registry
        - name: add constraint to start glance after keystone
          command: >
            pcs constraint order start openstack-keystone-clone
              then openstack-glance-registry-clone
        - name: enable glance resources
          command: pcs resource enable {{item}}
          with_items:
            - fs-varlibglanceimages-clone
            - openstack-glance-registry-clone
            - openstack-glance-api-clone
        - name: wait for glance to become active
          tags:
            - verify
          shell: |
            . /root/keystonerc_admin
            timeout 300 sh -c 'until glance image-list; do sleep 1; done'
    
## Update Cinder resources

Here we update cinder, adding a start constraint between cinder-api
and keystone.

We then enable all the cinder services and wait for cinder to become
active before continuing.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - cinder
      tasks:
        - name: update cinder resources
          shell: |
            pcs resource update {{item}} start-delay=
            pcs resource update {{item}} op monitor interval=60s
            [ "{{item}}" = "openstack-cinder-volume" ] ||
              pcs resource update {{item}} clone interleave=true
          with_items:
            - openstack-cinder-api
            - openstack-cinder-scheduler
            - openstack-cinder-volume
        - name: add constraint to start cinder after keystone
          command: >
            pcs constraint order start openstack-keystone-clone
              then openstack-cinder-api-clone
        - name: enable cinder resources
          command: pcs resource enable {{item}}
          with_items:
            - openstack-cinder-api-clone
            - openstack-cinder-scheduler-clone
            - openstack-cinder-volume
        - name: wait for cinder to become active
          tags:
            - verify
          shell: |
            . /root/keystonerc_admin
            timeout 300 sh -c 'until cinder list; do sleep 1; done'
    
## Update Nova resources

Here we update nova, adding a start constraint between nova-api
and keystone.

We then enable all the nova services and wait for nova to become
active before continuing.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - nova
      vars:
        nova_services:
            - openstack-nova-api
            - openstack-nova-consoleauth
            - openstack-nova-novncproxy
            - openstack-nova-conductor
            - openstack-nova-scheduler
      tasks:
        - name: update nova resources
          shell: |
            pcs resource update {{item}} start-delay=
            pcs resource update {{item}} op monitor interval=60s
            pcs resource update {{item}} clone interleave=true
          with_items: nova_services
        - name: add constraint to start nova after keystone
          command: >
            pcs constraint order start openstack-keystone-clone
              then openstack-nova-api-clone
        - name: enable nova resources
          command: pcs resource enable {{item}}-clone
          with_items: nova_services
        - name: wait for nova to become active
          tags:
            - verify
          shell: |
            . /root/keystonerc_admin
            timeout 300 sh -c 'until nova list; do sleep 1; done'
    
## Update Heat resources

Here we update heat, adding a start constraint between heat-api
and keystone.

We then enable all the heat services and wait for heat to become
active before continuing.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - heat
      tasks:
        - name: update heat resources
          shell: |
            pcs resource update {{item}} start-delay=
            pcs resource update {{item}} op monitor interval=60s
            pcs resource update {{item}} clone interleave=true
          with_items:
            - openstack-heat-api
            - openstack-heat-api-cfn
            - openstack-heat-api-cloudwatch
            - openstack-heat-engine
        - name: add constraint to start heat after keystone
          command: >
            pcs constraint order start openstack-keystone-clone
              then openstack-heat-api-clone
        - name: enable heat resources
          command: pcs resource enable {{item}}
          with_items:
            - openstack-heat-api-clone
            - openstack-heat-api-cfn-clone
            - openstack-heat-api-cloudwatch-clone
            - heat
        - name: wait for heat to become active
          tags:
            - verify
          shell: |
            . /root/keystonerc_admin
            timeout 300 sh -c 'until heat stack-list; do sleep 1; done'
    
## Update Apache (httpd) resource

Update and enable the `httpd` resource.  As of this writing we *do
not* wait for the service to become active, because determining the
proper URL for testing would be unnecessarily complicated.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - httpd
      tasks:
        - name: update httpd resources
          shell: |
            pcs resource update {{item}} start-delay=
            pcs resource update {{item}} op monitor interval=60s
            pcs resource update {{item}} clone interleave=true
          with_items:
            - httpd
        - name: enable httpd resources
          command: pcs resource enable {{item}}
          with_items:
            - httpd-clone
    
## Update Neutron resources

There are substantial changes in the Neutron resource configuration
between RHEL-OSP 5 and RHEL-OSP 6.  In particular, we replace the
`neutron-agents` resource group with individual clones for each
service.  We run Neutron in active/passive mode by (a) creating a
`neutron-scale` clone resource that can only run one instance at a
time (`clone_max=1`), and (b) adding constraints to the other
neutron services that tie them to whichever host is currently
running the `neutron-scale` instance.

After enabling all the Neutron resources, we first wait for the API
to become responsive, and then we wait explicitly for the L3 agent
to become active (because this agent can be slow to start and cause
erroneous failures of the post-check at the end of this playbook).


<!-- break -->

    - hosts: controllers[0]
      tags:
        - neutron
      tasks:
        # XXX: why does neutron have an explicit start timeout?
        - name: update neutron-server resource
          shell: |
            pcs resource update {{item}} start-delay=
            pcs resource update {{item}} op monitor interval=60s
            pcs resource update {{item}} op start timeout=60s
            pcs resource update {{item}} clone interleave=true
          with_items:
            - neutron-server
        - name: add constraint to start neutron after keystone
          command: >
            pcs constraint order start openstack-keystone-clone
              then neutron-server-clone
        - name: enable neutron-server resource
          command: pcs resource enable neutron-server-clone
    
        # Delete neutron-agents resource group, if it exists.
        - name: check if neutron-agents resource group exists
          command: crm_resource -q -r neutron-agents
          register: neutron_agents_group
          ignore_errors: true
        - name: delete neutron-agents resource group
          command: pcs resource delete neutron-agents
          when: neutron_agents_group|success
    
        # Delete individual agent resource definitions.
        - name: check if individual neutron agent resources exist
          command: crm_resource -q -r {{item}}
          with_items:
            - neutron-openvswitch-agent
            - neutron-dhcp-agent
            - neutron-l3-agent
            - neutron-metadata-agent
          register: neutron_agents
          ignore_errors: true
          when: neutron_agents_group|failed
        - name: delete individual neutron agent resources
          command: pcs resource delete {{item.item}}
          when: neutron_agents_group|failed and item.rc == 0
          with_items: neutron_agents.results
    
        # Delete additional neutron resources, if they exist.
        - name: check if other neutron resources exist
          command: crm_resource -q -r {{item}}
          with_items:
            - neutron-netns-cleanup
            - neutron-ovs-cleanup
            - neutron-scale-clone
          register: other_neutron
          ignore_errors: true
        - name: delete other neutron resources
          shell: pcs resource delete {{item.item}}
          when: item.rc == 0
          with_items: other_neutron.results
    
        # The neutron-scale resource is responsible for setting the
        # DEFAULT/host parameter in neutron configuration files.  We want
        # to first run it on all of our controllers, so we start it with
        # clone-max=(number of controllers), and then once it has run
        # everywhere we modify it with clone-max=1.
        - name: create neutron-scale resource
          command: >
            pcs resource create neutron-scale ocf:neutron:NeutronScale
              clone globally-unique=true clone-max={{groups.controllers|length}}
              interleave=true
        - name: wait for neutron-scale to start on all nodes
          shell: |
            while pcs status xml |
                xmllint --xpath '//clone[@id="neutron-scale-clone"]/resource[@active="false"]' -; do
              sleep 1
            done
        - name: make neutron-scale resource active/passive
          shell: |
            pcs resource disable neutron-scale-clone
            pcs resource meta neutron-scale-clone clone-max=1
            pcs resource enable neutron-scale-clone
    
Note that in the following tasks we are creating new resources
with `target-role=Stopped`.  This prevents Pacemaker from starting
them immediately; instead, we enable the services after setting
up all the order and colocation constraints.


<!-- break -->

        - name: create new neutron-ovs-cleanup resource
          command: >
            pcs resource create neutron-ovs-cleanup
              ocf:neutron:OVSCleanup clone interleave=true
              meta target-role=Stopped
        - name: create new neutron-netns-cleanup resource
          command: >
            pcs resource create neutron-netns-cleanup
              ocf:neutron:NetnsCleanup clone interleave=true
              meta target-role=Stopped
        - name: create new neutron agent resources
          command: >
            pcs resource create {{item}} systemd:{{item}} clone interleave=true
            meta target-role=Stopped
          with_items:
            - neutron-openvswitch-agent
            - neutron-dhcp-agent
            - neutron-l3-agent
            - neutron-metadata-agent
        # This forces all the neutron resources to colocate with
        # neutron-scale-clone.
        - name: configure neutron colocation contraints
          shell: |
            pcs constraint colocation add neutron-metadata-agent-clone with neutron-l3-agent-clone
            pcs constraint colocation add neutron-l3-agent-clone with neutron-dhcp-agent-clone
            pcs constraint colocation add neutron-dhcp-agent-clone with neutron-openvswitch-agent-clone
            pcs constraint colocation add neutron-openvswitch-agent-clone with neutron-netns-cleanup-clone
            pcs constraint colocation add neutron-netns-cleanup-clone with neutron-ovs-cleanup-clone
            pcs constraint colocation add neutron-ovs-cleanup-clone with neutron-scale-clone
        # This forces neutron resources to start after
        # neutron-scale-clone.
        - name: configure neutron ordering constraints
          shell: |
            pcs constraint order start neutron-scale-clone then neutron-ovs-cleanup-clone
            pcs constraint order start neutron-ovs-cleanup-clone then neutron-netns-cleanup-clone
            pcs constraint order start neutron-netns-cleanup-clone then neutron-openvswitch-agent-clone
            pcs constraint order start neutron-openvswitch-agent-clone then neutron-dhcp-agent-clone
            pcs constraint order start neutron-dhcp-agent-clone then neutron-l3-agent-clone
            pcs constraint order start neutron-l3-agent-clone then neutron-metadata-agent-clone
        - name: enable neutron resources
          command: pcs resource enable {{item}}
          with_items:
            - neutron-netns-cleanup-clone
            - neutron-ovs-cleanup-clone
            - neutron-openvswitch-agent-clone
            - neutron-dhcp-agent-clone
            - neutron-l3-agent-clone
            - neutron-metadata-agent-clone
        - name: wait for neutron to become active
          tags:
            - verify
          shell: |
            . /root/keystonerc_admin
            timeout 300 sh -c 'until neutron net-list; do sleep 1; done'
        # neutron-l3-agent is slow to start.  we wait for it here explicitly to
        # prevent erroneous failures of the post-check code at the end of the
        # playbook.
        - name: wait for neutron-l3-agent to become active
          tags:
            - verify
          shell: |
            . /root/keystonerc_admin
            timeout 300 sh -c 'until neutron agent-list | grep "L3 agent.*:-)"; do sleep 1; done'
    
## Restart compute services

This restarts OpenStack services on all of the compute hosts.


<!-- break -->

    - hosts: compute
      tags:
        - compute
      tasks:
        - name: restart all openstack services
          command: openstack-service start
    
## Post-check

This checks that there are no inactive resources at the conclusion
of the upgrade process.


<!-- break -->

    - hosts: controllers[0]
      tags:
        - post-check
      tasks:
        # make sure that everything that we disabled at the beginning of
        # this playbook has been re-enabled.
        - name: checking for inactive resources
          shell: |
            tmpfile=$(mktemp xmlXXXXXX)
            trap "rm -f $tmpfile" EXIT
    
            pcs status xml > $tmpfile
            xmllint --xpath '/crm_mon/resources/*/@id' $tmpfile | tr ' ' '\n'| cut -f2 -d'"' |
            while read id; do
              [ "$id" ] || continue
              if ! xmllint --xpath "/crm_mon/resources/*[@id='$id' and @active='true']|/crm_mon/resources/*[@id='$id']/*[@active='true']" $tmpfile > /dev/null 2>&1; then
                echo "$id: no active resources" >&2
                exit 1
              fi
            done
    
## Notes about this playbook

I've made heavy use of [xmllint][] in this playbook, in particular
its support for [xpath][] queries to parse the various XML output
available from Pacemaker commands.

[xmllint]: http://xmlsoft.org/xmllint.html
[xpath]: http://www.w3.org/TR/xpath/
