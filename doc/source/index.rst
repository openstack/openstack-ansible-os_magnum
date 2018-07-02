========================
OpenStack-Ansible Magnum
========================

Ansible role that installs and configures OpenStack Magnum. Magnum is
installed behind the Apache webserver listening on port 9511 by default.


To clone or view the source code for this repository, visit the role repository
for `os_magnum <https://github.com/openstack/openstack-ansible-os_magnum>`_.

Default variables
~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../defaults/main.yml
   :language: yaml
   :start-after: under the License.

Dependencies
~~~~~~~~~~~~

This role needs pip >= 7.1 installed on the target host.

To use this role, define the following variables:

.. code-block:: yaml

    # Magnum TCP listening port
    magnum_service_port: 9511

    # Magnum service protocol http or https
    magnum_service_proto: http

    # Magnum Galera address of internal load balancer
    magnum_galera_address: "{{ internal_lb_vip_address }}"

    # Magnum Galera database name
    magnum_galera_database_name: magnum_service

    # Magnum Galera username
    magnum_galera_user: magnum

    # Magnum rpc userid
    magnum_oslomsg_rpc_userid: magnum

    # Magnum rpc vhost
    magnum_oslomsg_rpc_vhost: /magnum

    # Magnum notify userid
    magnum_oslomsg_notify_userid: magnum

    # Magnum notify vhost
    magnum_oslomsg_notify_vhost: /magnum

This list is not exhaustive. See role internals for further details.

Wiring docker with cinder
~~~~~~~~~~~~~~~~~~~~~~~~~

If you need to use volumes, default_docker_volume_type should be set.
By default, Magnum doesn't need one.

To deploy Magnum with cinder integration, please set the following
in your ``/etc/openstack_deploy/user_variables.yml``:

.. code-block:: yaml

    magnum_config_overrides:
      cinder:
        default_docker_volume_type: lvm

If you have defined cinder_default_volume_type for all your nodes,
by defining it in your user_variables, you can re-use it directly:

.. code-block:: yaml

    magnum_config_overrides:
      cinder:
        default_docker_volume_type: "{{ cinder_default_volume_type }}"

Example playbook
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../examples/playbook.yml
   :language: yaml

Tags
~~~~

This role supports two tags: ``magnum-install`` and ``magnum-config``.
The ``magnum-install`` tag can be used to install and upgrade. The
``magnum-config`` tag can be used to maintain configuration of the
service.
