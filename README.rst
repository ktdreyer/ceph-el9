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

The thing that does this filtering is Pungi's "comps", `a large XML file <https://gitlab.com/redhat/centos-stream/release-engineering/comps/-/blob/main/comps-centos-stream-9.xml.in>`_ that lists all the leaf sub-packages that we want to put into RHEL 9.

This is done in `Trello <https://trello.com/b/wkDpptM1/ceph-el9>`_ now.

Step 4: build what we need in Copr
==================================

For the builds in RHEL 9: we need to download those from `CentOS 9's Koji
<https://kojihub.stream.centos.org/>`_ and rebuild them in our el9 Copr. In
parallel we need to petition to add those to CentOS 9's CRB repo.

For the builds not in RHEL 9, we need to clone those from Fedora and build them in the el9 Copr.

Walkthrough: Building for el9 from Fedora sources
-------------------------------------------------

Clone the fedora sources to your computer::

    fedpkg clone nasm
    cd nasm

Generate the SRPM from Fedora's sources::

    fedpkg srpm

Test a local mock build::

    mock -r el9-ceph-x86_64 nasm-2.15.03-5.fc35.src.rpm

Build the SRPM in Copr::

    copr-cli build ceph/el9 nasm-2.15.03-5.fc35.src.rpm

When the build completes, you'll see it listed in the `Copr web UI
<https://copr.fedorainfracloud.org/coprs/ceph/el9/builds/>`_.
