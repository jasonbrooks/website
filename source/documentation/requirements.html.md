---
author: amoralej
title: General purpose requirements
---

# Requirements management in RDO

1. toc
{:toc}

## Introduction

OpenStack services usually need some pieces of software which are not developed as
part of the project. They are are general purpose libraries (typically python
modules) or services used in some way to run or build OpenStack packages as databases, messaging brokers, etc...

OpenStack [requirements project](https://docs.openstack.org/developer/requirements/)
defines the policies and processes to manage requirements in upstream projects from
a global perspective.

## Managing OpenStack requirements in RDO

RDO provides all requirements for packaged services in RPM format from their own repos,
so that no software should be installed from external repositories. This packages can
be provided by:

- CentOS base repositories (base, updates and extras). This is the preferred source of
packages whenever possible.
- Other [CentOS SIG repositories](https://wiki.centos.org/SpecialInterestGroup) (Virtualization,
Storage, etc...). When a required package is being maintained by other CentOS SIG, it
will be reused for RDO repos.
- RDO CloudSIG repositories. When a package is not available from previous repos, it will
be provided in RDO repositores. Note that it's required that these packages exist previously
in Fedora so that they can be rebuilt with minimal changes (if any).

If you have questions or special requests, don't hesitate in contacting RDO using our
[mailing lists](/contribute/mailing-lists/) or #rdo channel in freenode.

### Adding a new requirement to RDO

When a new requirement is needed for an OpenStack project included in RDO, package maintainers
must follow this workflow:

![RDO dependencies](/images/new-dependencies.png)

Note that, typically new requirements are added only for the release of OpenStack under
development, not for stable releases, although they may be accepted in previous releases
if it's properly justified.

1. If the project follows global-requirements processes, make sure that the requirement has been
added to global-requirements.txt and upper-constraints.txt files as described in the [upstream
documentation](https://github.com/openstack/requirements/#proposing-changes)

    <br />
2. Check if the new requirement is present in CentOS base channels. The easiest way to do this
is using yum command from a system running CentOS 7:

        yum list "*<dependency>"

    If it's present, the desired package is already available to RDO users.

    <br />
3. If the package is not in CentOS base repos, you can check if it has been already built by
the CloudSIG using rdopkg:

        rdopkg info <package name>

    as, for example:

        $ rdopkg info python-eventlet
        1 packages found:

        name: python-eventlet
        project: python-eventlet
        conf: unmanaged-dependency
        patches: None
        distgit: https://github.com/rdo-common/python-eventlet.git
        buildsys-tags:
          cloud7-openstack-common-release: python-eventlet-0.17.4-4.el7
          cloud7-openstack-common-testing: python-eventlet-0.17.4-4.el7
          cloud7-openstack-newton-release: python-eventlet-0.18.4-2.el7
          cloud7-openstack-newton-testing: python-eventlet-0.18.4-2.el7
          cloud7-openstack-ocata-release: python-eventlet-0.18.4-2.el7
          cloud7-openstack-ocata-testing: python-eventlet-0.18.4-2.el7
          cloud7-openstack-pike-testing: python-eventlet-0.20.1-2.el7
          cloud7-openstack-pike-release: python-eventlet-0.20.1-2.el7
          cloud7-openstack-queens-release: python-eventlet-0.20.1-2.el7
          cloud7-openstack-queens-testing: python-eventlet-0.20.1-2.el7
          cloud7-openstack-rocky-testing: python-eventlet-0.20.1-2.el7

        master-distgit: https://github.com/rdo-common/python-eventlet.git
        review-origin: null
        review-patches: null
        tags:
          dependency: null
        maintainers:
        - apevec@redhat.com
        - hguemar@fedoraproject.org

    Note that the version of the package included in repositories is given by the
    CBS tags applied to each package (shown under buildsys-tags section for each package).
    Tags have a format cloud7-openstack-&lt;release>-&lt;phase> where:

    - release: is a the OpenStack release name, as pike, queens or rocky.
    - phase:
      - `candidate` phase is assigned to packages to be rebuilt in
      CBS but not pushed to any RDO repository.
      - `el7-build` (only available for Rocky and newer releases)
      is assigned to packages that only required to build other
      packages but are not a runtime requirement for any other package.
      - `testing` phase means that the package is used in deployments using RDO Trunk
      repo and published in a testing repo, but not official CloudSIG repository.
      - `release` phase means that is published in the official CloudSIG repository.
      This phase is only available after a RDO version has been officially released
      not for the one currently under development.

    For example, the package included in cloud7-openstack-queens-release will published
    in the [CloudSIG repo for queens](http://mirror.centos.org/centos-7/7/cloud/x86_64/openstack-queens/). The CBS tags flow will be:
    - *Runtime requirements:* candidate -> testing -> release
    - *Build requirements:* candidate -> el7-build

    Note that, for the release currently under development (rocky right now), only the
    testing phase will be available. The package included in cloud7-openstack-rocky-testing
    will be the one used to deploy from RDO Trunk Master repositories and it will be
    automatically pushed to cloud7-openstack-rocky-release at RDO Rocky is officially
    released and published.

    If the package is found for the required CBS tag, it's already in RDO repositories
    and no more actions are needed to add it to the repos.

    <br />
4. In case that the dependency is not in CentOS base or CloudSIG repo, you can check if it has been built
by other SIGs in [CBS web interface](http://cbs.centos.org/koji/). You can use wildcards in the packages
search expression. If you find the desired dependency, you can open a bug in [Red Hat Bugzilla for
RDO product](https://bugzilla.redhat.com/enter_bug.cgi?product=RDO&component=distribution) requesting
the inclussion of the package in RDO repos. RDO Core members will handle the request.

    <br />
5. If the new package is not in CBS, you must check if it's packaged in Fedora using the [package
browser](https://apps.fedoraproject.org/packages/). If the package exists, you need to open
a review to [rdoinfo project in RDO gerrit instance](https://review.rdoproject.org/r/#/q/project:rdoinfo)
adding the new dependency to `deps.yml` file as in [this example](https://review.rdoproject.org/r/#/c/13280/3/deps.yml):

        - project: sphinxcontrib-apidoc
         buildsys-tags:
           cloud7-openstack-rocky-candidate: python-sphinxcontrib-apidoc-0.2.1-6.el7
         name: python-sphinxcontrib-apidoc
         conf: fedora-dependency

     Where:
     - `project` and `name` must be the name of the main package (the same as in fedora).
     - `conf` must be `fedora-dependency`.
     - In `buildsys-tags` section a new line for the candidate tag in the OpenStack
  release in development (cloud7-openstack-rocky-candidate) with the required
  NVR (name-version-release) required, which must be the same one found in Fedora
  replacing fcXX part in release by el7. For example, for [python-sphinxcontrib-apidoc](https://apps.fedoraproject.org/packages/python-sphinxcontrib-apidoc/)
  the latest build is python-sphinxcontrib-apidoc-0.2.1-6.fc29, so in deps.yml
  cloud7-openstack-rocky-candidate must point point to python-sphinxcontrib-apidoc-0.2.1-6.el7.

    This review will rebuild the Fedora package in the CentOS Build System and make
  it available to be pushed to the next CBS phase.

    <br />
6. When the packages doesn't exist even in Fedora you need to add the package following the [New package
process](https://fedoraproject.org/wiki/New_package_process_for_existing_contributors). Note that a Fedora
packager needs to participate in this process. While RDO core members may maintain the new package for
common requirements used by different projects, dependencies for specific project must be maintained in
Fedora by the project team. Once the package is included in Fedora repos you can
create a gerrit review as explained in step 5.

    <br />
7. Once the package is rebuilt in CBS (review in step 5 is merged) you can push
it to the next phase, this means testing (for runtime dependencies) or el7-build
(for build requirements in rocky or newer releases). This is done by sending a new
review to [rdoinfo project](https://review.rdoproject.org/r/#/q/project:rdoinfo)
adding a new line under `buildsys-tags` to `deps.yml` file for the new tag as in
[this example](https://review.rdoproject.org/r/#/c/13469/1/deps.yml):

        buildsys-tags:
         cloud7-openstack-rocky-el7-build: python-sphinxcontrib-apidoc-0.2.1-6.el7

    Once this review is merged, the tag will be applied to this build and the package
 will added to the testing repo for rocky (note that some delay, up to 30 minutes
 is expected).

    <br />
8. After the package is available in the repos, you can add it to the list of *Requires* or *BuildRequires* in
your package spec file. Note that optional dependencies not used in default or common configurations should
not be added as Requires but installed only when needed.


### Updating a requirement in RDO CloudSIG repositories

There are some rules to follow when a requirement update is needed by a OpenStack project:

* If the dependency is included in upstream requirements project, the required version must be equal to
the version in [upper-constraints](https://github.com/openstack/requirements/blob/master/upper-constraints.txt) file.

* For packages installed from CentOS base repos, the package should be updated in CentOS/RHEL repos. This can
be requested opening a [bug in bugzilla for RHEL product](https://bugzilla.redhat.com/enter_bug.cgi?product=Red%20Hat%20Enterprise%20Linux%207).
This bug will be evaluated following the RHEL process.

* For packages installed from RDO CloudSIG repos, the package must be updated in Fedora first to the required
version. If it has not been updated first you can contact Fedora package maintainer or open a [bug for Fedora
product](https://bugzilla.redhat.com/enter_bug.cgi?product=Fedora). Once the package has been updated, yo can
send a [request to get it updated in RDO repos](https://bugzilla.redhat.com/enter_bug.cgi?product=RDO&component=distribution).

## Contact us

If you have questions or special requests about requirements, don't hesitate to contact RDO community members using our
[mailing lists](/contribute/mailing-lists/) or #rdo channel in freenode.
