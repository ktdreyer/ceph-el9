Building Ceph's dependencies for EPEL 9
=======================================

Our strategy is to fork packages from Fedora Rawhide into EPEL 9, like `we did
<https://github.com/ktdreyer/ceph-el8>`_ a few years ago for EPEL 8.

Step 1: Create a new Copr repo
==============================

DONE: https://copr.fedorainfracloud.org/coprs/ceph/el9

Step 2: Create a mock configuration that uses this Copr repo
============================================================

Place `<el9-ceph-x86_64.cfg>`_ into ``/etc/mock``. This relies on the `CentOS
Stream 9 config files
<https://github.com/rpm-software-management/mock/pull/751>`_ from
``mock-core-configs-34.5-1.fc34``

Step 3: Identify which builds already exist in RHEL 9
=====================================================

*"Wait, some of these builds already exist in RHEL 9? Why doesn't install-deps.sh find them?"*

The *build* exists, but the RHEL developers filter out certain sub-packages from the RHEL 9 product. For example, lua exists in RHEL 9, but lua-devel does not. The reason for this is that the RHEL developers only want to provide users with supported packages, and they don't want to provide packages that lead customers into unsupported configurations.

The thing that does this filtering is Pungi's configuration, `a large "comps" XML file <https://gitlab.com/redhat/centos-stream/release-engineering/comps/-/blob/main/comps-centos-stream-9.xml.in>`_, plus an `"additional_packages" config file <https://gitlab.com/redhat/centos-stream/release-engineering/pungi-centos/-/blob/centos-9-stream/shared/additional_and_filter_packages.conf>`_. These list all the leaf sub-packages that we want to put into RHEL 9.

This is done in `Trello <https://trello.com/b/wkDpptM1/ceph-el9>`_ now.

Step 4: build what we need in Copr
==================================

For the builds in RHEL 9: we need to download those from `CentOS 9's Koji
<https://kojihub.stream.centos.org/>`_ and rebuild them in our el9 Copr. In
parallel we need to petition to add those to CentOS 9's CRB repo.

For the builds not in RHEL 9, we need to clone those from Fedora and build them in the el9 Copr.

Walkthrough: Building for el9 from Fedora sources
-------------------------------------------------

Ensure your SSH public key is set up in `Fedora's Account System
<https://accounts.fedoraproject.org/>`_. Install ``fedpkg``, then clone a
package (eg. nasm) like so::

    fedpkg clone nasm
    cd nasm

Generate the SRPM from Fedora's sources::

    fedpkg srpm

Test a local mock build::

    mock -r el9-ceph-x86_64 nasm-2.15.03-5.fc35.src.rpm

(In my mock builds on an CentOS 8 Stream host in DreamCompute, ``mock`` hits a
bug where it cannot resolve any Yum repo host in DNS. ``mock
--isolation=simple`` works around this. TODO: reproduce without the Copr
config and report to mock team.)

Build the SRPM in Copr::

    copr-cli build ceph/el9 nasm-2.15.03-5.fc35.src.rpm

When the build completes, you'll see it listed in the `Copr web UI
<https://copr.fedorainfracloud.org/coprs/ceph/el9/builds/>`_.

Using this Copr repo
====================

On your RHEL 9 or CentOS 9 system::

    dnf copr enable -y ceph/el9

As developers change builds in this Copr rapidly, you can refresh your local
el9 system's cache::

    yum --disablerepo='*' --enablerepo=copr:copr.fedorainfracloud.org:ceph:el9 makecache

Backporting Ceph changes to pacific branch
==========================================

The following el9-related changes are in progress or merged to ``master`` in
Ceph upstream. We should backport these to the ``pacific`` branch in order to
support RHEL 9 for that release.

Required for building:

* https://tracker.ceph.com/issues/52471 - ``ceph.spec.in: drop gdbm from build deps``

* https://tracker.ceph.com/issues/52610 - ``cmake: link Threads::Threads
  instead of CMAKE_THREAD_LIBS_INIT`` (or workaround this by setting
  ``%bcond_with rbd_ssd_cache`` and ``%bcond_with rbd_rwl_cache``)

* On mem-constrainted builders (16GB), bump ``smp_limit_mem_per_job`` to
  ``3200000``

Important for testing upstream, not *strictly* required:

* https://tracker.ceph.com/issues/52449 - ``remove remaining references to lsb_release``
* https://tracker.ceph.com/issues/52472 - ``*: s/virtualenv/python -m venv/``
