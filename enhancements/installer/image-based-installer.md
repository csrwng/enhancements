---
title: image-based-installer
authors:
  - "@mresvanis"
  - "@eranco74"
reviewers:
  - "@patrickdillon"
  - "@romfreiman"
approvers:
  - "@zaneb"
api-approvers:
  - None
creation-date: 2024-04-30
last-updated: 2024-04-30
tracking-link:
  - https://issues.redhat.com/browse/MGMT-17600
see-also:
  - "/enhancements/agent-installer/agent-based-installer.md"
replaces: N/A
superseded-by: N/A
---

# Image-based Installer

## Summary

The Image-based Installer (IBI) is an installation method for on-premise
single-node OpenShift (SNO) clusters, that will use the following to run on the
hosts that are to become SNO clusters:

1. A seed [OCI image](https://github.com/opencontainers/image-spec/blob/main/spec.md)
   [generated](https://github.com/openshift-kni/lifecycle-agent/blob/main/docs/seed-image-generation.md)
   via the [lifecycle-agent](https://github.com/openshift-kni/lifecycle-agent)
   operator from a SNO system provisioned with the target OpenShift version.
2. A bootable installation ISO generated to provision multiple SNO clusters.
3. A configuration ISO generated to be used in a single SNO cluster installation.

Each of the aforementioned artifacts is potentially generated by a different
user persona. This enhancement focuses on the last two artifacts, which the
users will generate using a command-line tool. The installation ISO will be
configured with a [seed OCI image](https://github.com/openshift-kni/lifecycle-agent/blob/main/docs/seed-image-generation.md).
The latter is installed onto a SNO as a new [ostree stateroot](https://ostreedev.github.io/ostree/deployment/#stateroot-aka-osname-group-of-deployments-that-share-var)
and includes, among other files, the `/var`, `/etc` (with specific exclusions)
and `/ostree/repo` directories, which contain the target OpenShift version and
most of its configuration, but no site specific configuration and amounts
approximately to just over 1GB in size. The configuration ISO will contain the
site specific configuration data (e.g. cluster name, domain and crypto
objects), which need to be set up per cluster and are derived mainly from the
OpenShift installer [install config](https://github.com/openshift/installer/tree/release-4.15/pkg/asset/installconfig).

## Motivation

The primary motivation for relocatable SNO is the fast deployment of single-node
OpenShift. Telecommunications providers continue to deploy OpenShift at the Far
Edge. The acceleration of this adoption and the nature of existing
Telecommunication infrastructure and processes drive the need to improve
OpenShift provisioning speed at the Far Edge site and the simplicity of
preparation and deployment of Far Edge clusters, at scale.

IBI provides users with such speed and simplicity, but it currently needs the
[multicluster engine](https://docs.openshift.com/container-platform/4.15/architecture/mce-overview-ocp.html)
and/or the [Image-based Install operator](https://github.com/openshift/image-based-install-operator)
to generate the required installation and configuration artifacts. We would like
to enable users to generate the latter intuitively and independently, using
their own automation or even manual intervention to boot the host.

### User Stories

- As a user in a on-premise disconnected environment with no existing management
  cluster, I want to deploy a single-node OpenShift cluster using a [seed OCI image](https://github.com/openshift-kni/lifecycle-agent/blob/main/docs/seed-image-generation.md)
  and my own automation for provisioning.
- As a user in a on-premise disconnected environment with no existing management
  cluster, I want to generate a bootable installation ISO, common for a large
  number of hosts that are to be provisioned as single-node OpenShift
  clusters, using a [seed OCI image](https://github.com/openshift-kni/lifecycle-agent/blob/main/docs/seed-image-generation.md).
- As a user in a on-premise disconnected environment with no existing management
  cluster, I want to generate a configuration ISO for a specific host that is to
  be provisioned as a single-node OpenShift cluster using an already generated
  bootable installation ISO containing a [seed OCI image](https://github.com/openshift-kni/lifecycle-agent/blob/main/docs/seed-image-generation.md).

### Goals

- Install clusters with single-node topology.
- Install clusters in fully disconnected environments.
- Perform reproducible cluster builds from configuration artifacts.
- Require no machines in the cluster environment other than the one to be the
  single node of the cluster.
- Be agnostic to the tools used to provision machines, so that users can
  leverage their own tooling and provisioning.

### Non-Goals

- Replace any other OpenShift installation method in any capacity.
- Generate image formats other than ISO.
- Automate booting of the ISO image on the machines.
- Support installation configurations for cloud-based platforms.

## Proposal

A command-line tool will enable users to build a single custom RHCOS seed image
in ISO format, containing the components needed to provision multiple
single-node OpenShift clusters from that single ISO and multiple site
configuration ISO images, one per cluster to be installed.

The command-line tool will download the base RHCOS ISO, create an [Ignition](https://coreos.github.io/ignition/)
file with generic configuration data (i.e. configuration that is going to be
included in all clusters to be installed with that ISO) and generate an
image-based installation ISO. The Ignition file will configure the live ISO such
that once the machine is booted with the latter, it will install RHCOS to the
installation disk, mount the installation disk, restore the single-node
OpenShift from the [seed OCI image](https://github.com/openshift-kni/lifecycle-agent/blob/main/docs/seed-image-generation.md)
and optionally precache all release container images under the
`/var/lib/containers` directory.

The installation ISO approach is very similar to what is already implemented by
the functionality of the [Agent-based Installer](/enhancements/agent-installer-agent-based-installer.md), although
IBI differs from the OpenShift Agent-based Installer in several key aspects:

- while the Agent-based Installer may offer flexibility and versatility in certain scenarios,
  it may not meet the stringent time constraints and requirements of far-edge deployments
  in the telecommunications industry due to the inherently long installation process,
  exacerbated by low bandwidth and high packet latency.
- with the Agent-based Installer all cluster configuration needs to be provided upfront
  during the generation of the ISO image, while with IBI the cluster
  configuration is provided in an additional step.

IBI offers key advantages, where fast and reliable deployment at the edge is
crucial. By generating ISO images containing all the necessary components,
IBI significantly accelerates deployment times. Moreover, unlike the Agent-based
Installer, the image-based approach allows for cluster configuration to be
supplied upon deployment at the edge, rather than during the initial ISO
generation process. This flexibility enables operators to use a single generic
image for installing multiple clusters, streamlining the deployment process and
reducing the need for multiple customized ISO images.

The command-line tool will also support generating a configuration ISO with all
the site specific configuration data for the cluster to be installed provided as
input. The configuration ISO contents are the following:
- `ClusterInfo` (cluster name, base domain, hostname, node IP)
- SSH `authorized_keys`
- Pull Secret
- Extra Manifests
- Generated keys and certs (compatible with the generated admin `kubeconfig`)
- Static networking configuration

The site specific configuration data will be generated according to information
provided in the `install-config.yaml` and the manifests provided in the
installation directory as input. To complete the installation at the edge site:
- the cluster configuration for the edge location can be delivered by copying
  the config ISO content onto the node and placing it under `/opt/openshift/cluster-configuration/`.
- the cluster configuration can also be delivered using an attached ISO, a
  systemd service running on the host pre-installed and IBI will
  mount that ISO (identified by a known label) and copy the cluster configuration
  to `/opt/openshift/cluster-configuration/`.
- the cluster configuration data on the disk will be used to configure the
  cluster and allow OCP to start successfully.

### Workflow Description

The image-based installation high-level flow consists of the following
stages, each of which is performed by different users:

1. Generate a [seed OCI image](https://github.com/openshift-kni/lifecycle-agent/blob/main/docs/seed-image-generation.md)
   via the [Lifecycle Agent operator](https://github.com/openshift-kni/lifecycle-agent)
   and is set up with the desired OpenShift version. This OCI image will be used
   for multiple SNO cluster installations.
2. Generate a bootable installation ISO using the seed OCI image generated in
   the previous stage. This ISO will be also used for multiple SNO cluster
   installations.
3. Extract the installation ISO to the machine's disk at the factory.
4. Ship the machine to the edge site.
5. Generate a configuration ISO at the edge site, which will contain site
   specific configuration for a single SNO cluster installation.
6. Extract the configuration ISO contents under `/opt/openshift/cluster-configuration/`
   or attach the former (which a pre-installed systemd service will identify by
   a known label, mount and copy its contents to the aforementioned filesystem
   location), then boot the machine at the edge site to be provisioned as a SNO
   cluster.

### API Extensions

N/A

### Topology Considerations

#### Hypershift / Hosted Control Planes

N/A

#### Standalone Clusters

N/A

#### Single-node Deployments or MicroShift

The Image-based Installer targets single-node OpenShift deployments.

### Implementation Details/Notes/Constraints

Since we must allow users to provision hosts themselves, either manually or
using automated tooling of their choice, the ISO format offers the widest range
of compatibility. Building a single ISO to boot multiple hosts makes it
considerably easier for the user to manage. The additional site configuration
ISO is necessary for configuring each cluster securely and independently.

The user, before running the Image-based Installer, must generate a [seed OCI image](https://github.com/openshift-kni/lifecycle-agent/blob/main/docs/seed-image-generation.md)
via the [Lifecycle Agent SeedGenerator Custom Resouce (CR)](https://github.com/openshift-kni/lifecycle-agent/blob/main/docs/seed-image-generation.md).
The prerequisites to generating a seed OCI image are the following:

- an already provisioned single-node OpenShift cluster (seed SNO).
   - The CPU topology of that host must align with the target host(s), i.e. it
     should have the same number of cores and the same tuned performance
     configuration (ie. reserved CPUs).
- the [Lifecycle Agent](https://github.com/openshift-kni/lifecycle-agent/tree/main)
  operator must be installed on the seed SNO.

### Risks and Mitigations

N/A

### Drawbacks

N/A

## Open Questions [optional]

- Should the command-line tool that generates the installation ISO be a subcommand
  of the OpenShift installer, or a standalone binary?

  Having the functionality provided by the command-line tool in the OpenShift
  installer would be beneficial to the users, as the former refers to the
  provisioning of single-node OpenShift clusters and it should follow the
  OpenShift versions (i.e. the installation ISO generating tool for a specific
  OpenShift version should be used to install a seed OCI image with the same
  OpenShift version). It generates the required installation artifacts in the
  same way as the [Agent-based Installer](/enhancements/agent-installer/agent-based-installer.md)
  `openshift-install agent create image` command.

- Should the command-line tool that generates the configuration ISO be a subcommand
  of the OpenShift installer, or a standalone binary?

  Having the functionality provided by the command-line tool in the OpenShift
  installer would be a natural addition to the latter, as the former refers to
  the provisioning of single-node OpenShift clusters and consumes the OpenShift
  installer `install-config.yaml`. It generates the required installation
  artifacts in the same way as the [Agent-based Installer](/enhancements/agent-installer/agent-based-installer.md)
  `openshift-install agent create config-image` command.

## Test Plan

The Image-based Installer will be covered by end-to-end testing using virtual
machines (in a baremetal configuration), automated by some variation on the
metal platform [dev-scripts](https://github.com/openshift-metal3/dev-scripts/#readme).
This is similar to the testing of the Agent-based Installer, the baremetal IPI
and assisted installation flows.

## Graduation Criteria

**Note:** *Section not required until targeted at a release.*

TBD

### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s)

### Tech Preview -> GA

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing
- User facing documentation created in [openshift-docs](https://github.com/openshift/openshift-docs/)

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

### Removing a deprecated feature

N/A

## Upgrade / Downgrade Strategy

N/A

## Version Skew Strategy

### Configuration ISO and Seed OCI Image

The IBI configuration ISO contains the [cluster configuration file](https://github.com/openshift-kni/lifecycle-agent/blob/release-4.16/api/seedreconfig/seedreconfig.go).
The component responsible for parsing and using the latter is part of the seed
OCI image, which in turn is contained in the IBI installation ISO. That
component validates its compatibility with the cluster configuration file [version](https://github.com/openshift-kni/lifecycle-agent/blob/release-4.16/api/seedreconfig/seedreconfig.go#L6)
during the image-based installation. In the case of incompatibility between the
cluster configuration version and the seed OCI image version, the image-based
installation will fail with the respective error message.

### RHCOS ISO and Seed OCI Image

The RHCOS base ISO, which is contained in the IBI installation ISO and derived
from the OpenShift release image, has currently no strict requirements to be
tied to the seed OCI image OpenShift version. The features and configuration of
the underlying tools required to successfully complete an image-based
installation are Podman, SELinux and ostree. In order to remove the risk of
version skew between the RHCOS ISO and the seed OCI image, we plan on restricting
users to generating an installation ISO using the same RHCOS base ISO version as
the one contained in the seed OCI image OpenShift version.

### RHCOS ISO and Ignition

Since the IBI installation ISO is customized via an Ignition file, we need to
ensure that the RHCOS base ISO version is compatible with the latter. To that
end, the RHCOS base ISO will be fetched by looking into the [coreos stream metadata](https://github.com/openshift/installer/blob/master/data/data/coreos/rhcos.json)
embedded in the OpenShift installer binary and it will either be extracted from
the OpenShift release image, which contains the same version, or be downloaded
from the mirror URL in the metadata.

## Operational Aspects of API Extensions

N/A

## Support Procedures

N/A

## Alternatives (Not Implemented)

### Downloading the installation ISO from Assisted Installer SaaS

The Assisted Installer SaaS could be used to generate and serve the image-based
installation ISO, which would potentially enhance the user experience, compared
to configuring and executing a command-line tool. In addition, even in a fully
disconnected environment, the user could generate the ISO via the Assisted
Installer service and then carry it over to the disconnected environment.

However, for the Assisted Installer service this would be a completely new
scope, as the image-based installation ISO is not generated per cluster, but for
multiple SNO clusters with similar underlying hardware (due to the requirements
of the IBI flow) and its configuration input is different than what is currently
supported.

### Building the installation ISO in the Lifecycle Agent Operator

The Lifecycle Agent Operator could be used to generate and serve the image-based
installation ISO, as this is the component used to generate the seed OCI
image, which is the basis of the IBI flow.

However, the following reasons constitute a separate command-line tool a better
fit:

- the seed OCI image is generated by a different user persona (e.g.
  release/certification/other team) than the one to generate the image-based
  installation ISO.
- the same seed OCI image can be used to generate multiple image-based
  installation ISOs (e.g. with different installation disks).
- having 2 (installation ISO and configuration ISO) out of 3 (the 3rd is the
  seed OCI image) IBI artifacts generated by the same command-line tool
  simplifies the user experience.
- as a future enhancement we can provide generic seed OCI images, in order to skip
  the seed OCI image generation step altogether.

## Infrastructure Needed [optional]

N/A
