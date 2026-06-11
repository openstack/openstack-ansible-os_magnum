==========================
Magnum Cluster API drivers
==========================

.. warning::

   This documentation is relevant only for OpenStack-Ansible 2026.1 (Gazpacho)
   or later.
   An earlier version was provided in the `openstack-ansible-ops documentation <https://docs.openstack.org/openstack-ansible-ops/2025.2/mcapi.html>`_

About
-----

With deprecation of Heat driver during 2025.1 (Antelope) release, preliminary
Magnum deployment is assumed to be performed through one of the third-party
drivers.

At the moment there are at least 2 well-known Kubernetes Cluster API
drivers for the Magnum. This page describes the Vexxhost Magnum Cluster API
driver as well as Azimuth (HELM) Magnum Cluster driver.

Presence of a compatible "control plane" kubernetes cluster, which will be
responsible for spawning tenant clusters, is a pre-requirement for both of
these drivers. Such cluster is being prepared using `adriacloud.kubernetes <https://galaxy.ansible.com/ui/repo/published/adriacloud/kubernetes/>`_
collection during ``openstack.osa.setup_infrastructure`` playbook.

Same "control plane" cluster can be used by both of the drivers, so we consider
such kubernetes infrastructure as "shared" between drivers.

The following architectural features are present and are true for both drivers:

* The control plane k8s cluster is an integral part of the OpenStack-Ansible
  deployment, and forms part of the foundational components alongside MariaDB
  and RabbitMQ.
* The control plane k8s cluster is deployed on the infra hosts and integrated
  with the HAProxy load balancer and OpenStack internal API endpoint, and not
  exposed outside the deployment
* SSL is supported between all components and configuration is
  possible to support different certificate authorities on the internal
  and external load balancer endpoints.
* Control plane traffic can stay entirely within the management network
  if required.
* It is possible to do a completely offline install for airgapped environments


Highlevel overview of the Magnum infrastructure these playbooks will
build and operate against.

.. image:: assets/mcapi-architecture.png
    :scale: 80 %
    :alt: OSA Magnum Cluster API Architecture
    :align: center


Pre-requisites
--------------

* An existing OpenStack-Ansible deployment
* Control plane using LXC containers, bare metal deployment should work, but
  is not tested
* Core OpenStack services plus Octavia


Control Plane Kubernetes Cluster
--------------------------------

Adding Kubernetes cluster to inventory
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Alike to any other service, k8s cluster should be added to the
inventory first.

For that, define the physical hosts that will host the control plane k8s
cluster in `/etc/openstack_deploy/conf.d/k8s.yml`. This example is for
an all-in-one deployment and should be adjusted to match a real deployment
with multiple hosts if high availability is required.

.. literalinclude:: assets/k8s.yml
   :language: yaml

Kubernetes cluster configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can control various parameters of the deployment by defining overrides for
the control plane of the k8s cluster in
`/etc/openstack_deploy/group_vars/k8s_all/main.yml`.

.. literalinclude:: assets/k8s-config.yml
   :language: yaml


Deploying Kubernetes cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In order to deploy the Kubernetes cluster to control plane, run following
playbooks:

.. code-block:: bash

   openstack-ansible openstack.osa.containers_lxc_create --limit k8s_all,cluster_api_hosts
   openstack-ansible openstack.osa.k8s


Configuration of Cluster API drivers
------------------------------------

Vexxhost Cluster API driver
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The magnum-cluster-api driver for magnum can be found here:
https://github.com/vexxhost/magnum-cluster-api

Documentation for the Vexxhost magnum-cluster-api driver is here:
https://vexxhost.github.io/magnum-cluster-api/


You might need to define a series overrides for the magnum service itself and
the driver in particular. For that, you may create
`/etc/openstack_deploy/group_vars/magnum_all/main.yml` and place required
overrides in there.


Attention must be given to the SSL configuration. Users and workload clusters
will interact with the external endpoint and must trust the SSL certificate.
The magnum service and cluster-api can be configured to interact with either
the external or internal endpoint and must trust the SSL certificiate.
Depending on the environment, these may be derived from different certificate
authorities.

.. literalinclude:: assets/vexxhost-magnum-config.yml
   :language: yaml


Azimuth (HELM) Cluster API driver
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The Azimuth (HELM) Cluster API driver is developed under the OpenStack
governance, and the repository can be found here:
https://opendev.org/openstack/magnum-capi-helm

Please, check `project's documentation <https://docs.openstack.org/magnum-capi-helm/latest/user_docs/index.html>`_
for more details on the driver design and capabilities.

However, the driver is heavily depending on HELM charts, which are being
developed by Azimuth Cloud. The driver is depending on following HELM
charts:

  * `cluster-api-addon-provider <https://github.com/azimuth-cloud/cluster-api-addon-provider>`_
  * `capi-helm-charts <https://github.com/azimuth-cloud/capi-helm-charts>`_

There are several variables available for override with the driver. You can
place options to the `/etc/openstack_deploy/group_vars/magnum_all/main.yml`
file. A sample configuration file below:

.. literalinclude:: assets/azimuth-magnum-config.yml
   :language: yaml

Run the deployment
------------------

.. note::

   Please check :ref:`magnum-post-install` for details on how to upload required images
   and cluster templates before proceeding with deployment.

Deploy the magnum with a Cluster API driver

  .. code-block:: bash

     openstack-ansible openstack.osa.magnum


Optional Components
-------------------

Use of magnum-cluster-api-proxy with Vexxhost driver
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. note::

   The Magnum Cluster API Proxy service has been designed and tested for
   the Vexxhost CAPI driver.

As the control plane k8s cluster need to access a k8s control plane of tenant
cluster for it's further configuration, the only way to do it out of the box
is through the public network (Floating IP). This means, that API of the k8s
control plane must be globally reachable, which posses a security threat to
such tenant clusters.

On order to solve the issue and provide access for the control plane k8s
cluster to tenant clusters inside their internal networks a proxy service
is introduced.

Proxy service must be spawned on hosts, where Neutron Metadata agents are
spawned. For LXB/OVS these are members of ``neutron-agent_hosts``, while
for OVN the service should be installed to all ``compute_hosts``
(or ``neutron_ovn_controller``).

The service will configure own HAProxy instance and create backends
for managed k8s clusters to point inside corresponding network
namespaces.
Service does not spawn own namespaces, but leverages already existing
metadata namespaces to get connection to the Load Balancer inside
the project network.

Configuration of the service is relatively trivial:

   .. code-block:: yaml

      # Define a group of hosts where to install the service.
      # OVN: compute_hosts / neutron_ovn_controller
      # OVS/LXB: neutron_metadata_agent
      magnum_capi_vexxhost_proxy_hosts: compute_hosts
      # Define address and port HAProxy instance to listen on
      magnum_capi_vexxhost_proxy_environment:
         PROXY_BIND: "{{ management_address }}"
         PROXY_PORT: 44355

Also, in case of proxy service deployment, ensure that variable
``magnum_capi_vexxhost_git_install_branch`` is defined for the
``magnum_capi_vexxhost_proxy_hosts`` as well, or align value of the
``magnum_capi_vexxhost_git_install_branch`` with
``magnum_capi_vexxhost_proxy_install_branch`` to avoid conflicts caused by
different versions of driver used.

Once configuration is complete, you can run the playbook:

   .. code-block:: bash

      openstack-ansible openstack.osa.magnum_capi_proxy


Troubleshooting
---------------

Local testing
-------------

An OpenStack-Ansible All-in-One configured with Magnum and Octavia is
capable of running a functioning magnum-cluster-api deployment.

Sufficient memory should be available beyond the minimum 8G usually required
for an All-in-One. A multi-node workload cluster may require nova to boot
several Ubuntu images in addition to an Octavia load balancer instance. 64G
would be an appropriate amount of system RAM.

There also must be sufficient disk space in `/var/lib/nova/instances` to
support the required number of instances - the normal minimum of 60G
required for an All-in-One deployment will be insufficient, 500G would
be plenty.
