==============================
Configuring HAProxy (optional)
==============================

HAProxy provides load balancing services and SSL termination when hardware
load balancers are not available for high availability architectures deployed
by OpenStack-Ansible. The default HAProxy configuration provides highly-
available load balancing services via keepalived if there is more than one
host in the ``haproxy_hosts`` group.

.. important::

  Ensure you review the services exposed by HAProxy and limit access
  to these services to trusted users and networks only. For more details,
  refer to the :dev_docs:`Securing network access to OpenStack services <reference/architecture/security.html#securing-network-access-to-openstack-services>` section.

.. note::

  For a successful installation, you require a load balancer. You may
  prefer to make use of hardware load balancers instead of HAProxy. If hardware
  load balancers are in use, then implement the load balancing configuration for
  services prior to executing the deployment.

To deploy HAProxy within your OpenStack-Ansible environment, define target
hosts to run HAProxy:

   .. code-block:: yaml

       haproxy_hosts:
         infra1:
           ip: 172.29.236.101
         infra2:
           ip: 172.29.236.102
         infra3:
           ip: 172.29.236.103

There is an example configuration file already provided in
``/etc/openstack_deploy/conf.d/haproxy.yml.example``. Rename the file to
``haproxy.yml`` and configure it with the correct target hosts to use HAProxy
in an OpenStack-Ansible deployment.

Making HAProxy highly-available
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If multiple hosts are found in the inventory, deploy
HAProxy in a highly-available manner by installing keepalived.

Edit the ``/etc/openstack_deploy/user_variables.yml`` to skip the deployment
of keepalived along HAProxy when installing HAProxy on multiple hosts.
To do this, set the following:

.. code-block:: yaml

   haproxy_use_keepalived: False

To make keepalived work, edit at least the following variables
in ``user_variables.yml``:

.. code-block:: yaml

   haproxy_keepalived_external_vip_cidr: 192.168.0.4/25
   haproxy_keepalived_internal_vip_cidr: 172.29.236.54/16
   haproxy_keepalived_external_interface: br-flat
   haproxy_keepalived_internal_interface: br-mgmt

- ``haproxy_keepalived_internal_interface`` and
  ``haproxy_keepalived_external_interface`` represent the interfaces on the
  deployed node where the keepalived nodes bind the internal and external
  vip. By default, use ``br-mgmt``.

- On the interface listed above, ``haproxy_keepalived_internal_vip_cidr`` and
  ``haproxy_keepalived_external_vip_cidr`` represent the internal and
  external (respectively) vips (with their prefix length).

- Set additional variables to adapt keepalived in your deployment.
  Refer to the ``user_variables.yml`` for more descriptions.

To always deploy (or upgrade to) the latest stable version of keepalived.
Edit the ``/etc/openstack_deploy/user_variables.yml``:

.. code-block:: yaml

   keepalived_use_latest_stable: True

The HAProxy nodes have group vars applied that define the configuration
of keepalived. This configuration is stored in
``group_vars/haproxy_all/keepalived.yml``. It contains the variables
needed for the keepalived role (master and backup nodes).

Keepalived pings a public IP address to check its status. The default
address is ``193.0.14.129``. To change this default,
set the ``keepalived_ping_address`` variable in the
``user_variables.yml`` file.

.. note::

   The keepalived test works with IPv4 addresses only.

You can adapt keepalived to your environment by either using our override
mechanisms (per host with userspace ``host_vars``, per group with
userspace``group_vars``, or globally using the userspace
``user_variables.yml`` file)

Configuring keepalived ping checks
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

OpenStack-Ansible configures keepalived with a check script that pings an
external resource and uses that ping to determine if a node has lost network
connectivity. If the pings fail, keepalived fails over to another node and
HAProxy serves requests there.

The destination address, ping count and ping interval are configurable via
Ansible variables in ``/etc/openstack_deploy/user_variables.yml``:

.. code-block:: yaml

   keepalived_ping_address:         # IP address to ping
   keepalived_ping_count:           # ICMP packets to send (per interval)
   keepalived_ping_interval:        # How often ICMP packets are sent

By default, OpenStack-Ansible configures keepalived to ping one of the root
DNS servers operated by RIPE. You can change this IP address to a different
external address or another address on your internal network.

Securing HAProxy communication with SSL certificates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The OpenStack-Ansible project provides the ability to secure HAProxy
communications with self-signed or user-provided SSL certificates. By default,
self-signed certificates are used with HAProxy. However, you can
provide your own certificates by using the following Ansible variables:

.. code-block:: yaml

    haproxy_user_ssl_cert:          # Path to certificate
    haproxy_user_ssl_key:           # Path to private key
    haproxy_user_ssl_ca_cert:       # Path to CA certificate

Refer to `Securing services with SSL certificates`_ for more information on
these configuration options and how you can provide your own
certificates and keys to use with HAProxy. User provided certificates should
be folded and formatted at 64 characters long. Single line certificates
will not be accepted by HAProxy and will result in SSL validation failures.
Please have a look here for information on `converting your certificate to
various formats <https://search.thawte.com/support/ssl-digital-certificates/index?page=content&actp=CROSSLINK&id=SO26449>`_.
If you want to use `LetsEncrypt SSL Service <https://letsencrypt.org/>`_
you can activate the feature by providing the following configuration in
``/etc/openstack_deploy/user_variables.yml``. Note that this requires
that ``external_lb_vip_address`` in
``/etc/openstack_deploy/openstack_user_config.yml`` is set to the
external DNS address.

.. code-block:: yaml

   haproxy_ssl_letsencrypt_enable: true
   haproxy_ssl_letsencrypt_email: example@example.com

.. warning::

   There is no certificate distribution implementation at this time, so
   this will only work for a single haproxy-server environment.  The
   renewal is automatically handled via CRON and currently will shut
   down haproxy briefly during the certificate renewal.  The
   haproxy shutdown/restart will result in a brief service interruption.

.. _Securing services with SSL certificates: https://docs.openstack.org/project-deploy-guide/openstack-ansible/draft/app-advanced-config-sslcertificates.html

Configuring additional services
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Additional haproxy service entries can be configured by setting
``haproxy_extra_services`` in ``/etc/openstack_deploy/user_variables.yml``

For more information on the service dict syntax, please reference
``playbooks/vars/configs/haproxy_config.yml``

An example HTTP service could look like:

.. code-block:: yaml

    haproxy_extra_services:
      - service:
          haproxy_service_name: extra-web-service
          haproxy_backend_nodes: "{{ groups['service_group'] | default([]) }}"
          haproxy_ssl: "{{ haproxy_ssl }}"
          haproxy_port: 10000
          haproxy_balance_type: http
          # If backend connections should be secured with SSL (default False)
          haproxy_backend_ssl: True
          haproxy_backend_ca: /path/to/ca/cert.pem
          # Or if certificate validation should be disabled
          # haproxy_backend_ca: False

Additionally, you can specify haproxy services that are not managed
in the Ansible inventory by manually specifying their hostnames/IP Addresses:

.. code-block:: yaml

    haproxy_extra_services:
      - service:
          haproxy_service_name: extra-non-inventory-service
          haproxy_backend_nodes:
            - name: nonInvHost01
              ip_addr: 172.0.1.1
            - name: nonInvHost02
              ip_addr: 172.0.1.2
            - name: nonInvHost03
              ip_addr: 172.0.1.3
          haproxy_ssl: "{{ haproxy_ssl }}"
          haproxy_port: 10001
          haproxy_balance_type: http


Adding additional global VIP addresses
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In some cases, you might need to add additional internal VIP addresses
to the load balancer front end. You can use the HAProxy role to add
additional VIPs to all front ends by setting them in the
``extra_lb_vip_addresses`` variable.

The following example shows extra VIP addresses defined in the
``user_variables.yml`` file:

.. code-block:: yaml

   extra_lb_vip_addresses:
     - 10.0.0.10
     - 192.168.0.10

Adding Access Control Lists to HAProxy front end
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Adding ACL rules in HAProxy is easy. You just need to define haproxy_acls and
add the rules in the variable

Here is an example that shows how to achieve the goal

.. code-block:: yaml


   - service:
          haproxy_service_name: influxdb-relay
          haproxy_acls:
              write_queries:
                 rule: "path_sub -i write"
              read_queries:
                 rule: "path_sub -i query"
                 backend_name: "influxdb"

This will add two acl rules ``path_sub -i write`` and ``path_sub -i query``  to
the front end and use the backend specified in the rule. If no backend is specified
it will use a default ``haproxy_service_name`` backend.
