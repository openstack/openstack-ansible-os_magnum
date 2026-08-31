.. _building-azimuth-images:

=============================================
Building Kubernetes images for Azimuth driver
=============================================

The Magnum Azimuth Cluster API driver boot cluster
nodes from a pre-built Kubernetes image rather than installing packages
at runtime. This page shows how to build that image with the
`Kubernetes SIGs image-builder <https://github.com/kubernetes-sigs/image-builder>`_
project's QEMU/OpenStack target, and upload it to Glance for use in a
Magnum cluster template.

Prerequisites
~~~~~~~~~~~~~

The build uses QEMU/KVM, so the host needs the relevant packages
installed (most bare-metal and virtualized hosts support this out of
the box). On Ubuntu:

.. code-block:: shell-session

   # apt install qemu-kvm qemu-utils unzip
   # sudo usermod -a -G kvm <yourusername>
   # sudo chown root:kvm /dev/kvm

Log out and back in for the group change to apply. Packer, the Goss
provisioner plugin, and Ansible are also required.

Image Builder requires ``ansible-core`` 2.18.18 or later. Distribution
packages are usually well behind this, so install ``ansible-core`` with
``pip`` instead:

.. code-block:: shell-session

   $ pip3 install --user "ansible-core==2.18.18"

Building the image
~~~~~~~~~~~~~~~~~~

.. code-block:: shell-session

   # git clone https://github.com/kubernetes-sigs/image-builder
   # cd image-builder/images/capi
   # make deps-qemu
   # make build-qemu-ubuntu-2404

Ubuntu 22.04, 24.04, and 26.04 targets are available out of the box —
use ``build-qemu-ubuntu-2204``, ``-2404``, or ``-2604``(each also has
an ``-efi`` variant).
The resulting ``qcow2`` is written to
``images/capi/output/BUILD_NAME+kube-KUBERNETES_VERSION`` and can be
uploaded to Glance:

.. code-block:: shell-session

   # openstack image create k8s-1.36.1-ubuntu-2404 \
       --disk-format qcow2 --container-format bare \
       --property os_distro=ubuntu \
       --property kube_version=v1.36.1 \
       --file /path/to/output/ubuntu-2404-kube-v1.36.1/ubuntu-2404-kube-v1.36.1

Other distros are available for the QEMU target too, just swap the
suffix on ``make build-qemu-<distro>``:

* ``ubuntu-2204`` / ``-2204-efi`` / ``-2404`` / ``-2404-efi`` /
  ``-2604`` / ``-2604-efi``
* ``rhel-9`` (needs ``RHSM_USER``/``RHSM_PASS`` for the subscription)
* ``rockylinux-9`` (and ``rockylinux-9-cloudimg``)
* ``centos-9``

Pinning the Kubernetes version
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The default version tracks ``n-2`` and lives in
``images/capi/packer/config/kubernetes.json``. Don't edit that file -
override it with Packer variables instead:

* ``kubernetes_semver`` — e.g. ``v1.30.5``
* ``kubernetes_series`` — minor series, e.g. ``v1.30`` (selects the
  ``pkgs.k8s.io`` repo)
* ``kubernetes_deb_version`` — e.g. ``1.30.5-1.1`` (Ubuntu/Debian)
* ``kubernetes_rpm_version`` — e.g. ``1.30.5`` (RHEL/CentOS/Rocky)

Only the ``deb`` or ``rpm`` variant matching your target OS is strictly
required, but setting all four keeps builds consistent. Pass them
inline via ``PACKER_FLAGS``:

.. code-block:: shell-session

   # PACKER_FLAGS="--var 'kubernetes_semver=v1.30.5' \
       --var 'kubernetes_series=v1.30' \
       --var 'kubernetes_deb_version=1.30.5-1.1' \
       --var 'kubernetes_rpm_version=1.30.5'" \
     make build-qemu-ubuntu-2404

or via a JSON file with ``PACKER_VAR_FILES``:

.. code-block:: shell-session

   # cat > k8s-version.json <<EOF
   {
     "kubernetes_semver": "v1.30.5",
     "kubernetes_series": "v1.30",
     "kubernetes_deb_version": "1.30.5-1.1",
     "kubernetes_rpm_version": "1.30.5"
   }
   EOF
   # PACKER_VAR_FILES=k8s-version.json make build-qemu-ubuntu-2404

The chosen version string must exist in the ``pkgs.k8s.io`` repo for
that series, check ``https://pkgs.k8s.io/core:/stable:/<series>/deb/``
(or ``/rpm/``) first, or the build will fail on package install.

See the upstream `OpenStack provider docs
<https://image-builder.sigs.k8s.io/capi/providers/openstack>`_ for
other providers and build options, including remote builds on an
OpenStack cloud instead of a local KVM host.
