.. _magnum-post-install:

Post-deployment
---------------

Deploying the magnum service makes the API components available to use.
Additional configuration is required to make a working Kubernetes cluster,
including loading the correct Image and setting up a suitable Cluster Template.

This example is intended to show the steps required and should be updated
as needed for the version of k8s and associated components.

Images
~~~~~~

All drivers in Magnum have a series of requirements, and all of them require a
specifically prepared image for the k8s cluster control plane and workers.

However, these images are not interoperable, so each driver requires it's own
image. The image will be matched to the driver by the ``os_distro`` property
or ``magnum_driver`` property if defined.

So the first step is to obtain the correct image matching your driver and
upload it to the Glance.
You can rely on ``magnum_glance_images`` variable for the image to be uploaded
or do this step manually.

Heat driver
^^^^^^^^^^^

Heat driver relies on a fedora-coreos image. This can be done either
manually or using variables within the os_magnum role.

Manual upload:

.. code-block:: bash

    wget https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/38.20230806.3.0/x86_64/fedora-coreos-38.20230806.3.0-openstack.x86_64.qcow2.xz
    xz -d fedora-coreos-38.20230806.3.0-openstack.x86_64.qcow2.xz
    openstack image create "fedora-coreos-latest" --disk-format raw --container-format bare \
      --file fedora-coreos-38.20230806.3.0-openstack.x86_64.qcow2.xz --property os_distro='fedora-coreos'

Via os_magnum playbooks and data in user_variables.yml

.. code-block:: yaml

   magnum_glance_images:
    - name: fedora-coreos-latest
      disk_format: qcow2
      image_format: bare
      visibility: public
      compressed_format: xz
      url: https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/38.20230806.3.0/x86_64/fedora-coreos-38.20230806.3.0-openstack.x86_64.qcow2.xz
      properties:
      os_distro: "fedora-coreos"
      checksum: "1bbf0707a518f514c478d78f1b96d0f8"
      checksum_compressed: "da359b10f9aa165c4f81e6cd9ca5f81b"
      hide_method: community
      keep_copies: 1

Vexxhost driver
^^^^^^^^^^^^^^^

Vexxhost driver does use specially prepared Ubuntu image for tenant's cluster
workers and control plane.

Such image can be built using `diskimage-builder <https://docs.openstack.org/diskimage-builder/>`
with a custom element Vexxhost provides: `capo-image-elements <https://github.com/vexxhost/capo-image-elements>`.

Please refer to the ``capo-image-elements`` README file for more details on
how to build a compatible image.

Unlike to the Heat driver, you will need to have a separate image per supported
Kubernetes version.

You can also use a pre-built version, which is mainly designed for testing
only.
For that, add the following record to the
`/etc/openstack_deploy/group_vars/magnum_all/main.yml` file:


.. code-block:: yaml

    magnum_glance_images:
      - name: "ubuntu-22.04-v1.34.6"
        url: "https://github.com/vexxhost/capo-image-elements/releases/download/2026.04-2/ubuntu-22.04-v1.34.6.qcow2"
        disk_format: qcow2
        visibility: public
        properties:
          os_distro: ubuntu
          architecture: "x86_64"
          os_type: linux
          hw_disk_bus: scsi
          hw_scsi_model: virtio-scsi
          hw_qemu_guest_agent: "no"



Azimuth (HELM) driver
^^^^^^^^^^^^^^^^^^^^^

Azimuth driver takes simmilar approach to the Vexxhost, and also relies on a
specifically prepared Ubuntu image for client's workers and control plane.

But unlike the Vexxhost, image build process is performed by `packer <https://image-builder.sigs.k8s.io/capi/capi.html>`_

You can also use images built by Azimuth Cloud from their `Azimuth image releases <https://github.com/stackhpc/azimuth-images/releases/latest>`_
You can take URI of the image from the ``manifest.json file``.

.. note::

    We define ``magnum_driver`` in image properties explicitly, as both Vexxhost and Azimuth
    drivers are satisfying ``os_distro: ubuntu`` criteria


.. code-block:: yaml

    magnum_glance_images:
      - name: "ubuntu-22.04-v1.34.8"
        url: "https://azimuth-images.stackhpc.cloud/ubuntu-jammy-kube-v1.34.8-260518-1604.qcow2"
        disk_format: qcow2
        visibility: public
        properties:
          os_distro: ubuntu
          architecture: "x86_64"
          os_type: linux
          hw_disk_bus: scsi
          hw_scsi_model: virtio-scsi
          kube_version: v1.34.8
          magnum_driver: k8s_capi_helm_v1

Cluster templates
~~~~~~~~~~~~~~~~~

.. note::

    OpenStack-Ansible deploys the Magnum API service. It is not in scope
    for OpenStack-Ansible to maintain a guaranteed working cluster template as this
    will vary depending on the precise version of Magnum deployed and the required
    version of k8s and it's dependencies.

Heat driver
^^^^^^^^^^^

Manual configuration:

.. code-block:: bash

    openstack coe cluster template create <name> --coe kubernetes --external-network <ext-net> \
        --image "fedora-coreos-latest" --master-flavor <flavor> --flavor <flavor> --master-lb-enabled \
        --docker-volume-size 50 --network-driver calico --docker-storage-driver overlay2 \
        --volume-driver cinder \
        --labels boot_volume_type=<your volume type>,boot_volume_size=50,kube_tag=v1.18.6,availability_zone=nova,helm_client_url="https://get.helm.sh/helm-v3.4.0-linux-amd64.tar.gz",helm_client_sha256="270acb0f085b72ec28aee894c7443739271758010323d72ced0e92cd2c96ffdb",helm_client_tag="v3.4.0",etcd_volume_size=50,auto_scaling_enabled=true,auto_healing_enabled=true,auto_healing_controller=magnum-auto-healer,etcd_volume_type=<your volume type>,kube_dashboard_enabled=True,monitoring_enabled=True,ingress_controller=nginx,cloud_provider_tag=v1.19.0,magnum_auto_healer_tag=v1.19.0,container_infra_prefix=<docker-registry-without-rate-limit> -f yaml -c uuid

The equivalent Cluster Template configuration through os_magnum and data in
user_variables.yml

.. code-block:: yaml

    magnum_cluster_templates:
      - name: <name>
        coe: kubernetes
        external_network_id: <network-id>
        image_id: <image-id>
        master_flavor_id: <master-flavor-id>
        flavor_id: <minon-flavor-id>
        master_lb_enabled: true
        docker_volume_size: 50
        network_driver: calico
        docker_storage_driver: overlay2
        volume_driver: cinder
        labels:
          boot_volume_type: <your volume type>
          boot_volume_size: 50
          calico_tag: v3.26.4
          container_runtime: containerd
          containerd_version: 1.6.31
          containerd_tarball_sha256: 75afb9b9674ff509ae670ef3ab944ffcdece8ea9f7d92c42307693efa7b6109d
          cloud_provider_tag: v1.27.3
          cinder_csi_plugin_tag: v1.27.3
          k8s_keystone_auth_tag: v1.27.3
          magnum_auto_healer_tag: v1.27.3
          octavia_ingress_controller_tag: v1.27.3
          kube_tag: v1.28.9-rancher1
          availability_zone: nova
          helm_client_url: "https://get.helm.sh/helm-v3.4.0-linux-amd64.tar.gz"
          helm_client_sha256: "270acb0f085b72ec28aee894c7443739271758010323d72ced0e92cd2c96ffdb"
          helm_client_tag: v3.4.0
          etcd_volume_size: 50
          auto_scaling_enabled: true
          auto_healing_enabled: true
          auto_healing_controller: magnum-auto-healer
          etcd_volume_type: <your volume type>
          kube_dashboard_enabled: True
          monitoring_enabled: True
          ingress_controller: octavia
          container_infra_prefix: <docker-registry-without-rate-limit>


It will be necessary to specify a docker registry (potentially hosting your own
mirror or cache) which does not enforce rate limits when deploying Magnum in a
production environment.

Vexxhost driver
^^^^^^^^^^^^^^^

The minimal working cluster template at the moment of writing for the Vexxhost
CAPI driver can be defined like this:

.. code-block:: yaml

    magnum_cluster_templates:
        - name: <name>
          coe: kubernetes
          dns_nameserver: '8.8.8.8'
          external_network_id: 'public'
          flavor_id: "m1.medium"
          image_id: "ubuntu-22.04-v1.34.6"
          labels:
            kube_tag: "v1.34.6"
            octavia_provider: amphora
          master_flavor_id: "m1.medium"
          master_lb_enabled: "True"
          network_driver: "calico"


Azimuth (HELM) driver
^^^^^^^^^^^^^^^^^^^^^

The minimal working cluster template at the moment of writing for the Vexxhost
CAPI driver can be defined like this:

.. code-block:: yaml

    magnum_cluster_templates:
      - name: <name>
        cluster_distro: ubuntu
        coe: kubernetes
        docker_storage_driver: overlay2
        external_network_id: public
        flavor_id: m1.medium
        floating_ip_enabled: True
        image_id: "ubuntu-22.04-v1.34.8"
        insecure_registry: "localhost:5000"
        labels:
          boot_volume_size: 30
          cloud_provider_enabled: "True"
          csi_cinder_availability_zone: nova
          kube_dashboard_enabled: "False"
          kube_tag: "v1.34.8"
          calico_tag: "v3.31.4"
          octavia_provider: "amphora"
          octavia_lb_algorithm: SOURCE_IP_PORT
        master_flavor_id: m1.medium
        master_lb_enabled: true
        network_driver: calico
        public: true
        registry_enabled: true
        server_type: vm
        state: present
        volume_driver: cinder
        dns_nameserver: 8.8.8.8


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
