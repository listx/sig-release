# Packaging Kubernetes <!-- omit in toc -->

_Author(s): Sumitran Raghunathan ([@sumitranr](https://github.com/sumitranr))_
_Editor(s): Stephen Augustus ([@justaugustus](https://github.com/justaugustus))_
_Original document: [Building debs/rpms for Kubernetes
](https://docs.google.com/document/d/1PAN5tVJO_NMfHZmnk8mDQTwAbFHPky7JBgWJgckNjro/edit?usp=sharing)_

- [Introduction](#introduction)
- [Communication](#communication)
- [Release Steps](#release-steps)
  - [Permissions](#permissions)
  - [Clone Release Repository](#clone-release-repository)
  - [Authenticate](#authenticate)
  - [Build the Debs & RPMs](#build-the-debs--rpms)
    - [Notes](#notes)
  - [Validating packages](#validating-packages)
  - [Package verification tests](#package-verification-tests)

## Introduction

This guide outlines the process of building debs/rpms for Kubernetes minor and patch releases.

**DISCLAIMER: This document was initially drafted for Google internal release managers and has been published here (with edits) for the sake of visibility into the Kubernetes build/packaging process. With that in mind, there are several links and processes referenced here that can only be accessed/performed by a Google employee with the requisite access to Google build systems/tooling. We will refer to these Googlers as ["Kubernetes Build Admins"][kubernetes-build-admins] across this guide.**

## Communication

The [Patch Release Team][patch-release-team] (patch releases) or [Branch Manager][branch-manager-handbook] (minor releases) will reach out to the [Kubernetes Build Admins][kubernetes-build-admins] via [email][kubernetes-build-admins] or `#release-managers` on [Kubernetes Slack](https://slack.k8s.io) (for more synchronous communication) requesting help to build the debs and rpms.

Patch Release Team members or Branch Managers requesting debs/rpms should be sure to provide explicit details on the package name(s), version(s), and revision(s) they need built.

**n.b. As much as possible, communications with the Kubernetes Build Admins should happen on public forums (email, Slack public channels), not direct messaging, except in instances where doing so would run contrary to our [security policies][security-release-process] e.g., building packages for a release which addresses an embargoed CVE.**

## Release Steps

In this process, we are pulling the artifacts published by a Release Manager to the GCS bucket and building debs/rpms to be published to the rapture repository.

Refer to the [Branch Manager handbook][branch-manager-build-and-release] for details on the artifacts that are built by the Branch Manager.

See also the [rapture documentation][rapture-readme], which contains details on running `rapture` (Google internal packaging tool).

### Permissions

Must be a member of mdb group - [mdb/cloud-kubernetes-release](mdb/cloud-kubernetes-release) to be able to perform the debs/rpms releases.


### Clone Release Repository

Before proceeding with the release, ensure the [kubernetes/release][kubernetes/release] repository is checked out.

**NOTE**: It is a good idea to start in a clean directory, when possible.

_The directory name can be anything. We'll use `$HOME/k8s-1.20.0` as the example here._

```shell
mkdir -p $HOME/k8s-1.20.0
cd $HOME/k8s-1.20.0
git clone https://github.com/kubernetes/release.git
cd release
```

### Authenticate

Run the following commands to ensure that we are logged in and also the proper project context is setup.


```shell
prodaccess
gcloud auth login
gcloud config set project kubernetes-release-test
```

### Build the Debs & RPMs

The entire build process takes several hours. Once you are ready to begin, the debs and rpms can be built using [rapture][rapture].

`rapture` can be executed as follows:

```shell
/google/src/head/depot/google3/experimental/users/mehdy/kubernetes/k8s-rapture.sh <version> # <version> should be the Kubernetes version we are building the debs/rpms for e.g., `1.20.0`
```

#### Notes
- There are several points during the process where you will be prompted to answer “y/N” or your password.
- There will be a warning about trusty. This can be ignored.


### Validating packages

Now that `rapture` has successfully complete, we need to verify the packages that were just created. This validation can be done on any instance where Kubernetes is not already installed.

**If you are on a system with any of these packages are already installed, you must uninstall them first.**

Follow the [kubeadm instructions][kubeadm-install] to install kubeadm, kubelet, and kubectl.

```shell
# <version> should be the Kubernetes version we are building the debs/rpms for e.g., `1.20.0`
version=
[[ -n "${version}" ]] \
    && curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg \
    | sudo apt-key add - \
    && echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" \
    | sudo tee /etc/apt/sources.list.d/kubernetes.list \
    && sudo apt-get update -q \
    && sudo apt-get install -qy kubelet="${version}-00" kubectl="${version}-00" kubeadm="${version}-00"
```

### Package verification tests

In addition to manual validation, we also have some periodic package verification tests that run on prow.

See the [Release Engineering dashboard][release-engineering-dashboard] on Testgrid.
The following jobs are currently configured to do some aspect of package validation:
- https://testgrid.k8s.io/sig-release-misc#debian-unstable
- https://testgrid.k8s.io/sig-release-misc#periodic-packages-pushed
- https://testgrid.k8s.io/sig-release-misc#periodic-packages-install-deb


**These tend to break when we are in the middle of a push.**

If any of these tests are broken, the [Patch Release Team][patch-release-team] should receive an alert regarding the failure.

If there is continued test failure on this dashboard without intervention from the Patch Release Team, escalate to the current [Release Team][release-team] and [test-infra on-call][test-infra-oncall].


[branch-manager-handbook]: /release-team/role-handbooks/branch-manager/README.md
[branch-manager-build-and-release]: /release-team/role-handbooks/branch-manager/README.md#build-and-release
[kubeadm-install]: https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl
[kubernetes/release]: https://git.k8s.io/release
[kubernetes-build-admins]: https://groups.google.com/forum/#!forum/kubernetes-build-admins
[patch-release-team]: /releases/patch-releases.md
[rapture]: https://cs.corp.google.com/piper///depot/google3/experimental/users/mehdy/kubernetes/k8s-rapture.sh
[rapture-readme]: https://g3doc.corp.google.com/cloud/kubernetes/g3doc/release/rapture.md?cl=head
[release-engineering-dashboard]: https://testgrid.k8s.io/sig-release-misc
[release-team]: https://groups.google.com/forum/#!forum/kubernetes-release-team
[security-release-process]: /security-release-process-documentation/security-release-process.md
[test-infra-oncall]: https://go.k8s.io/oncall
