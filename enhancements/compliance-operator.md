---
title: OpenShift Compliance operator
authors:
  - "@jhrozek"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2019-10-22
last-updated: 2019-10-22
status: provisional
---

# OpenShift Compliance operator

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift/docs]

## Summary

The Compliance operator provides a way to scan nodes in an OKD cluster
to asses compliance against a given standard (for example FedRAMP moderate)
and in a future version, remediate the findings to get the cluster into
a compliant state. The scan is performed by the OpenSCAP tool.
OpenSCAP is well-established, [standards compliant](https://www.open-scap.org/features/standards/)
and [NIST-certified](https://csrc.nist.gov/projects/security-content-automation-protocol-validation-pr/validated-products-and-modules/142-red-hat-scap-1-2-product-validation-record)
project that administrators are aware of and trust.

For the first iteration, the operator would only provide a report with
the findings and proposed remediations which the cluster administrator
can then apply on their own. Remediation would be developed in one
of the future versions.

## Motivation

The default installation of OKD is not compliant (and is not expected
to be, as some requirements have impact on performance or flexibility of use
of the cluster) with different standards users in regulated environments
must adhere to.

Enabling users to be compliant with standards their respective industry
requires would enable OKD to be used in those environments. This is especially
true of the US public sector.

### Dependencies
As said above, the operator would orchestrate OpenSCAP scans on cluster
nodes. However, there are several areas where OpenSCAP needs to be extended
to be better usable for scanning an OKD cluster:

* OpenSCAP did not support merging of results from several nodes into a single
  bundle. Each scan on each node produces a separate report. There is an
  RFE filed against OpenSCAP and planned by the OpenSCAP team to aggregate
  results from all the cluster nodes into a single report.
* The remediations were typically done using a shell script or Ansible. For
  OpenShift we’d want the remediations done using MachineConfigs or other
  Custom Resources. However, the remediations are not a hard prerequisite for
  the first version of the operator, even identifying the gaps in compliance
  would be useful.
* For the gaps in compliance we identify, a check must be written. These
  checks are being compiled as a parallel effort. Having a full set of
  checks and/or remediations is not a hard prerequisite for the operator
  itself, though as the content would be delivered separately.

These have all been filed with the OpenSCAP team and acknowledged by them.

### Goals

A cluster administrator is able to use the operator to assess the degree
of compliance against a given security standard. In particular this would
mean:

* Provide a way for cluster administrator to start cluster scans
* Inspect the scan result on the high level (pass/fail/error)
* Provide a way to collect the detailed results of the scan
  * The results would list a way to remediate the gaps, typically by the means
    of deploying certain manifests to OKD (e.g. a Machineconfig), a different
    way of changing cluster configuration or even a free-form recommendation text.

### Non-Goals

The operator itself does not perform any scans or remediations, only
wraps the OpenSCAP scanner and provides an OKD-native API to
run the scans.

## Proposal

An instance of a scan would be represented by a CRD. The CRD would include
the scan profile (e.g. FedRAMP moderate), the content to scan against
(typically `ocp4`) and optionally a set of individual rules (e.g. is `auditd`
enabled on the nodes?). The CRD would be processed by the operator, which
would launch a pod on each node to scan that node and deliver scan results.

The pod would consist of two containers - a scanner and a log collector.
This is because OpenSCAP writes the report to a file, so the scanner and
the collector share a directory where the scanner writes the report, the
collector would wait for the report to be created and write the contents
of that file to a location where the cluster administrator can grab the
report and inspect it.

The scanner container in the pod would mount the node's file system into
a directory and point the `openscap-chroot` binary there.

### User Stories

#### As an OKD admin, I want to assess the degree of compliance of my cluster with FedRAMP moderate.

This is a use-case where the admin just wants to visualize the gaps in
compliance. They would perform the remediation themselves, potentially
based on the MCOs the compliance operator suggests.

### Implementation Details

There [exists a PoC](https://github.com/jhrozek/openscap-operator) of the implementation.

Some things to note from the PoC implementation are:
 * The operator itself schedules a pod per node in the cluster. This was selected
   (instead of e.g. DaemonSet) because the pods don't have to be long-lived and
   can just exit when the scan is done.
 * The collector container in the pod reads the contents of the scan results
   from a file produced by the scanner container and uploads them to a ConfigMap.
   This is perhaps a bit hacky, but most volume types do not support
   ReadWriteMany access mode and we need to get the results from files in
   N pods across N nodes. We could mount a per-pod volume and then gather
   the results into another volume when the scan is done, but this still
   means we need a post-run gather operation.
 * For viewing the results by administrator might use [a script](https://github.com/jhrozek/scapresults-k8s/blob/master/scapresults/fetchresults.py)

### Risks and Mitigations
 * [jhrozek]: Not sure what to write here. The scan might be
   resource-intensive perhaps? The scanner containers need to mount the
   host fs? The permissions need careful review?


## Design Details

The `compliance-operator` would use the following API to track a compliance scan status:

### API Specification
The proposed compliance scan API instance

```
apiVersion: complianceoperator.compliance.openshift.io/v1alpha1
kind: ComplianceScan
metadata:
  name: example-scan
spec:
  profile: xccdf_org.ssgproject.content_profile_ospp
  content: ssg-ocp4-ds.xml
```

The profiles and content would be provided as part of the container image. This
way, we can ensure that the auditor is using officially supported content.

## Need Feedback

How we present the remediations and allow the cluster administrator to execute
them is something that should be discussed more.

On a high level, there will be three kinds of remediations:
 * General free-form text guidances. For example, your IDP must support 2FA.
 * More specific advise, but still needs to be applied manually. For example,
   configure a message of the day so that a legal notice gets displayed after
   login
 * Gaps that can be remediated in a completely autonomous fashion. For example
   make sure that the `auditd` service is enabled.

At first, we would like to display the remediation in the HTML report so that
the administrator can copy the advise and, if it's possible to apply directly,
do that.

However, when it comes to applying the remediation, there are two options:

* Output the remediations to yaml files that the user can download and apply.
  The caveat of this is that it makes it cumbersome: First check the report,
  then download the remediations, then apply them manually.

* Add the capability so that the opreator can schedule a workload that will
  apply these remediations. The workflow would be as follows: Check the report
  then modify the CRD so that the remediation is applied. This is more
  automatic and user-friendly, however, it requires us giving a lot of
  privileges to the workload, which might not be ideal.


### Test Plan

**Note:** *Section not required until targeted at a release.*

TBD

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

TBD

### Upgrade / Downgrade Strategy

TBD


### Version Skew Strategy

TBD

## Implementation History

TBD

## Drawbacks

TBD

## Alternatives

Provide a "cookbook" of recipes that provide the cluster administrator
with a list of things to check and configure so that their cluster
is compliant.
