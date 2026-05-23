========================
OpenStack-Ansible Magnum
========================

Ansible role that installs and configures OpenStack Magnum. Magnum is
installed behind the Apache webserver listening on port 9511 by default.

To clone or view the source code for this repository, visit the role repository
for `os_magnum <https://github.com/openstack/openstack-ansible-os_magnum>`_.

.. note::

   Heat driver has been deprecated by Magnum team in 2025.1 (Epoxy) release
   and is completely removed in 2026.2 (Hibiscus) release.

   All new deployments should define ``magnum_k8s_driver`` to ``vexxhost`` or
   ``azimuth`` and avoid ``heat`` driver if possible.

Design and Configuration
~~~~~~~~~~~~~~~~~~~~~~~~

.. toctree::
   :maxdepth: 1

   magnum-capi
   post-deployment

Default variables
~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../defaults/main.yml
   :language: yaml
   :start-after: under the License.


Example playbook
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../examples/playbook.yml
   :language: yaml

Tags
~~~~

In addition to generic OpenStack-Ansible tags, this role supports two tags:

  * ``magnum-install`` - can be used to install and upgrade the service
  *  ``magnum-config`` - can be used to maintain configuration of the
                         service
