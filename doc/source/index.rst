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

    # Magnum rabbit userid
    magnum_rabbitmq_userid: magnum

    # Magnum rabbit vhost
    magnum_rabbitmq_vhost: /magnum

This list is not exhaustive. See role internals for further details.

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
