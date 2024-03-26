# OpenStackVersion

## Overview

This document describes how to use the OpenStackVersion
CR to apply a minor updates to a deployed OpenStack
installation (including both Controlplane and Dataplane
Services)

## Requirements

 * Use of OLM (Operator Lifecycle Manager) is assumed
 * The CSV (ClusterServiceVersion) for the openstack-operator
   is used as the primary resource for versioning and the
   associated ENV variables for OpenStack service containers.
   These are typically prefixed with RELATED\_IMAGE...
   The CSV defines the version.
 * OpenStack operators are always upgraded first via OLM. This
   can be either via manual or automatic. Upgrading operators
   will trigger OpenStackVersion to provide access to a new
   available version.
 * And ENV called OPENSTACK\_VERSION must be set in the CSV
   for openstack-operator.  This controls the 'availableVersion'
   of OpenStackVersion.
   

## General Update Workflow

OpenStack is installed by following the normal installation procedure. When
an OpenStackControlplane resource is created in any given namespace an
OpenStackVersion resource will be immediately created when the OpenStackControlPlane
is reconciled (1 OpenStackVersion per namespace).

When a new set of Operators is installed via OLM (either manually or automatically)
the OpenStackVersion resources will reconcile and a new 'AvailableVersion' will
be set on each CR.

In order to apply an update an administrator sets the 'TargetVersion' to the
same value as the 'AvailableVersion' on the OpenStackVersion resources. This
triggers the OpenStackVersion to reconcile and will update the images
on the associated OpenStackControlplane and OpenStackDataplane resources in
that namespace.

```shell
$ oc get openstackversion
NAME              TARGET VERSION   AVAILABLE VERSION
openstack         1.0.0            1.0.1
```

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackVersion
metadata:
  labels:
  name: openstack
spec:
  openstackControlPlaneName: openstack
  targetVersion: 1.0.1
```

## Custom images for Cinder Volume

In order to allow administrators to configure customized Cinder volume backends
with custom containerImages for each backend you must specify those images
from automatic updates. This is because there is no way to fully automate
the creation of those images at this time. To configure a custom
Cinder volume named 'netapp' updates you can add the name 'netapp' to the
'cinderVolumeCustomImages' map on the OpenStackVersion CR and configure
a custom image for that volume backend. Multiple named Cinder volume backends may
be configured in this manner.

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackVersion
metadata:
  labels:
  name: openstack
spec:
  openstackControlPlaneName: openstack
  targetVersion: 1.0.1
  cinderVolumeCustomImages:
    netapp: <custom image URL location>
```

## Custom Images for Neutron

TBD

## Custom images for other OpenStack services

In general administrators should not need to customize containerImages
for any other OpenStack services images. The exception would be
the need to apply a hotfix. In order to accommodate a hotfixed or
patched containerImage you can update the 'serviceCustomContainerImages'
map on the OpenStackVersion to contain the camel cased
name for the services that needs to be customized
with a custom container image. For example if you need
to hotfix the Glance API service container image you could
add 'GlanceApi' and the custom container image to the
'serviceCustomContainerImages' map.

```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackVersion
metadata:
  labels:
  name: openstack
spec:
  openstackControlPlaneName: openstack
  targetVersion: 1.0.1
  serviceCustomContainerImages:
    GlanceApi: <custom image URL location>
```

## Custom ordering of service updates

Container image updates currently occur in parallel for a given CR.
First OVN will be updated on the Dataplane CRs. Then the Controlplane will be updated.
Finally the rest of the Dataplane CRs container images will be updated.

If a situation arises where we need to update one service before
another we can use OpenStackVersion to implement this in the future.

## Things currently not supported

 * Rolling back to a prior openstack version directly via OpenStackVersion.
   A rollback would require the administrator to install an old CSV,
   and patch various CRs to update the containerImages among other things.
 * Major upgrades are not targeted at this time. However the OpenStackVersion
   CR may be useful as a means to help drive those as well.
