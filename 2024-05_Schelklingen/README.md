# Hack The Garden 05/2024 Wrap Up

## 🗃️ OCI Helm Release Reference For `ControllerDeployment`s

**Problem Statement:** Today, `ControllerDeployment`s contain the base64-encoded, gzip'ed, tar'ed raw Helm chart inside [their specification](https://github.com/gardener/gardener/blob/master/example/25-controllerdeployment.yaml#L11). Such an API blows up the backing ETCD unnecessarily, and it is error-prone/cumbersome to maintain.

**Motivation/Benefits**: 🔧 Reduced operational complexity, 🔄 enabled reusability for other scenarios.

**Achievements:** The `core.gardener.cloud/v1` API has been introduced for the `ControllerDeployment` which provides a more mature API and also supports OCI repository-based Helm chart references. It is also more extensible (hence, it could also support other deployment options in the future, e.g., `kustomize`). Based on this foundation, it is now possible to specify the URL of an OCI repository containing the Helm chart. `gardenlet`'s `ControllerInstallation` controller fetches the Helm chart from there and installs it as usual.

**Next Steps:** Currently, we don't cache the downloaded OCI Helm charts (i.e., in every reconciliation, we pull them again). This might need to get optimized to keep network traffic under control. Unit tests have to be written for the OCI registry puller, and the PR has to be opened.

**Issue:** https://github.com/gardener/gardener/issues/9773

**Code/Pull Requests**: https://github.com/stackitcloud/gardener/tree/controllerdeployment, https://github.com/gardener/gardener/pull/9771

<hr />

## 👨🏼‍💻 `gardener-operator` Local Development Setup With `gardenlet`s

**Problem Statement:** Today, there are two development setups for Gardener: The first, most common one is based on Gardener's [control plane Helm chart](https://github.com/gardener/gardener/tree/master/charts/gardener/controlplane) and other [custom manifests](https://github.com/gardener/gardener/tree/master/example/gardener-local/etcd) to bring up Gardener and a seed cluster. The second one is using [`gardener-operator`](https://github.com/gardener/gardener/blob/master/docs/concepts/operator.md) and the [`operator.gardener.cloud/v1alpha1.Garden`](https://github.com/gardener/gardener/blob/master/example/operator/20-garden.yaml) resource to bring up a full-fledged garden cluster. However, this setup does not bring up a seed cluster, hence, creating a `Shoot` is not possible. Generally, it would be better if we could harmonize the scenarios such that we only have to maintain one solution.

**Motivation/Benefits**: 👨🏼‍💻 Improved developer productivity, 🧪 increased output qualification.

**Achievements:** A new Skaffold configuration has been created which deploys `gardener-operator` and its `Garden` CRD, and later the `gardenlet` which register the garden cluster as a seed cluster. This also includes the basic Gardener configuration (`CloudProfile`s, `Project`s, etc.) and the registration of the `provider-local` extension. With this setup, it is now possible to create `Shoot`s as well.

**Next Steps:** Optimizing the readiness probes to speed up the reconciliation times should be considered.

**Code/Pull Requests**: https://github.com/gardener/gardener/pull/9763

<hr />

## 👨🏻‍🌾 Extensions For Garden Cluster Via `gardener-operator`

**Problem Statement:** A Gardener installation usually needs additional and tedious preparation tasks to be done outside "the Kubernetes world", e.g. creating storage buckets for ETCD backups or managing DNS records for the API server and the ingress controller. All of those could actually be automated via `gardener-operator` to reduce operational complexity. These tasks even overlap with requirements that have already been implemented for shoot clusters by Gardener extensions, i.e., the code is already available and could be reused.

**Motivation/Benefits**: 🔧 Reduced operational complexity, ✨ support more use cases/scenarios, 📦 provide out-of-the-box solution for Gardener community.

**Achievements:** The `Garden` controller has been augumented to deploy `extensions.gardener.cloud/v1alpha1.{BackupBucket,DNSRecord}` resources as part of its reconciliation flow. A new `operator.gardener.cloud/v1alpha1.Extension` CRD has been introduced to register extensions on `gardener-operator` level (specification is similar to `Controller{Registration,Deployment}`s. Several new controllers have been added to reconcile the new CRDs - the concepts are very similar to what already happens for extensions in the seed cluster and the related existing code in `gardenlet`. In case the garden cluster is a seed cluster at the same time, multiple instances of the same extension are needed. This requires that we prevent simultaneous reconciliations of the same extension object by different extension controllers. For this purpose, a `class` field has been added to the extension APIs, and extensions can be configured accordingly to restrict their watches for only objects of a specific `class`.

**Next Steps:** Deployment of extension admission components is still missing. Also, validation of the `operator.gardener.cloud/v1alpha1.Extension` as well as tests and documentation is missing. In the future, all `BackupBucket` and `DNSRecord` extension controllers must be adapted such that they support the scenario of running in the garden cluster.

**Issue:** https://github.com/gardener/gardener/issues/9635

**Code/Pull Requests**: https://github.com/metal-stack/gardener/commits/hackathon-gardener-operator-extensions/

<hr />

## 🪄 Gardenlet Self-Upgrades For Unmanaged `Seed`s

**Problem Statement:** In order to keep `gardenlet`s in unmanaged seeds up-to-date (i.e., in seeds which are no shoot clusters, like the "root cluster"), its Helm chart must be regularly deployed to it. This requires network connectivity to such clusters which can be challenging if they reside behind a firewall or in restricted environments. Similar challenges might arise for the to-be-developed [autonomous shoot clusters](https://github.com/gardener/gardener/issues/2906) (see also the [topic summary](https://github.com/gardener-community/hackathon/blob/main/2023-05_Leverkusen/README.md#-bootstrapping-masterful-clusters-aka-autonomous-shoots) from one of the previous Hackathons). It would be much simpler if `gardenlet` could keep itself up-to-date, based on configuration read from the garden cluster.

**Motivation/Benefits**: 🔧 Reduced operational complexity, ✨ support more use cases/scenarios.

**Achievements:** A new `seedmanagement.gardener.cloud/v1alpha1.Gardenlet` resource has been introduced whose specification is similar to the `seedmanagement.gardener.cloud/v1alpha1.ManagedSeed` resource. It allows specifying deployment values (replica count, resource requests, etc.) as well as the `gardenlet`'s component configuration (feature gates, seed spec, etc.). In addition, the `Gardenlet` object must contain a URL to an OCI registry storing `gardenlet`'s Helm chart. A new controller within `gardenlet` watches such resources, and if needed, downloads the Helm chart and applies it with the provided configuration to its own cluster.

**Next Steps:** Write unit tests and documentation (including a guide that can be followed when a `gardenlet` needs to be deployed to a new unmanaged soil/seed cluster). Open pull request.

**Code/Pull Requests**: https://github.com/metal-stack/gardener/commits/hackathon-gardenlet-self-upgrade/

<hr />

## 🦺 Type-Safe Configurability In `OperatingSystemConfig` For `containerd`, DNS, NTP, etc.

**Problem Statement:** Some Gardener extensions have to manipulate the `OperatingSystemConfig` for shoot worker nodes by changing `containerd` configuration, DNS or NTP servers, or similar. Currently, the API does not support first-class fields for such operations. Providing Bash scripts that apply the respective configs as part of `systemd` units is the only option. This makes the development process rather tedious and error-prone.

**Motivation/Benefits**: 👨🏼‍💻 Improved developer productivity.

**Achievements:** The `OperatingSystemConfig` API has been augmented to support the `containerd`-config related use-cases existing today. `gardener-node-agent` has been adapted to evaluate the new fields and apply the respective configs. Managing DNS and/or NTP servers probably needs to be handled by the OS extensions directly so that this information is already available during machine bootstrapping (i.e., before `gardener-node-agent` starts). The `os-gardenlinux`, `runtime-gvisor`, and `registry-cache` extensions have been adapted to the newly introduced `containerd`-config API. This allowed deleting a large portion of Bash scripts and related `systemd` units or `DaemonSet`s manipulating the host file system. Note that tackling this entire topic only became possible because we have developed [`gardener-node-agent`](https://github.com/gardener/gardener/blob/master/docs/concepts/node-agent.md), [an achievement from one of the previous Hackathons](https://github.com/gardener-community/hackathon/blob/main/2023-05_Leverkusen/README.md#%EF%B8%8F-introduction-of-gardener-node-agent).

**Next Steps:** The API adaptations in `gardener/gardener` have to be finalized first. This includes adding unit tests, augmenting the extension developer documentation, and polishing the code. Once merged and released, the work in the extensions can continue. The DNS/NTP server configuration requirements need to be discussed separately since the concept above does not fit well.

**Issue:** https://github.com/gardener/gardener/issues/8929

**Code/Pull Requests**: https://github.com/metal-stack/gardener/tree/enh.osc-api, https://github.com/gardener/gardener-extension-os-gardenlinux/pull/169, https://github.com/Gerrit91/gardener-extension-runtime-gvisor/tree/hackathon-improved-osc-api, https://github.com/timuthy/gardener-extension-registry-cache/tree/enh.osc-registry-poc

<hr />

## 👮 Expose Shoot API Server In [Tailscale](https://tailscale.com/) VPN

**Problem Statement:** The most common ways to secure a shoot cluster is to [apply ACLs](https://github.com/stackitcloud/gardener-extension-acl), or to use an `ExposureClass` which exposes the API server only within a corporate network. However, managing the ACL configuration can become difficult with a growing number of participants (needed IP addresses), especially in a dynamic environment and work-from-home scenarios. `ExposureClass`es might be not possible because no corporate network might be available. A Tailscale-based VPN, however, is a scalable and manageable alternative.

**Motivation/Benefits**: 🛡️ Increased cluster security, ✨ support more use cases/scenarios.

**Achievements:** A document has been compiled which explains what a shoot owner can do to achieve exposing their API server within a Tailscale VPN. Writing an extension or any code does not make sense for this topic. For each Tailnet, only one API server can be exposed.

**Next Steps:** The documentation shall be published on https://gardener.cloud and submitted to the Tailscale newsletter (they are calling for content/success stories).

**Code/Pull Requests**: https://gardener.cloud/docs/guides/administer-shoots/tailscale/

<hr />

## ⌨️ Rewrite [gardener/vpn2](https://github.com/gardener/vpn2) From Bash To Golang

**Problem Statement:** Currently, the VPN components mostly consist out of Bash scripts which are hard to maintain and easy to break (since they are untested). [Similar to how we have refactored other Bash scripts in Gardener to Golang](https://github.com/gardener-community/hackathon/blob/main/2023-05_Leverkusen/README.md#%EF%B8%8F-introduction-of-gardener-node-agent), we could increase developer productivity by eliminating the scripts in favor of testable Golang code.

**Motivation/Benefits**: 👨🏼‍💻 Improved developer productivity, 🧪 increased output qualification.

**Achievements:** All functionality has been rewritten to Golang, whereby all integration scenarios with Gardener have been considered. The validation and tests with both the local and an OpenStack-based environments were successful. The pull requests (already including unit tests) have been opened for `gardener/vpn2` and the integration in `gardener/gardener`.

**Next Steps:** The documentation needs to be adapted. Minor cosmetics and cleanups have to be performed. For the future, it should be considered to move the new Golang code to `gardener/gardener`. This could enable Gardener e2e tests of the VPN-related code (currently, a new image/release of `gardener/vpn2` is a prerequisite).

**Code/Pull Requests**: https://github.com/gardener/vpn2/pull/84, https://github.com/gardener/gardener/pull/9774

<hr />

## 🕳️ Pure IPv6-Based VPN Tunnel

**Problem Statement:** Today, the shoot pod, service and node network CIDRs may not overlap with the hard-coded VPN network CIDR. Hence, users are limited to use certain ranges (even if they would like to have another network design). Instead of making the VPN network CIDR configurable on seed level, a proper solution is to switch the VPN tunnel to a pure IPv6-based network which can transport both IPv4 and IPv6 traffic. This is a more mature solution because this lifts the mention CIDR restriction for IPv4 scenarios. For IPv6, the restriction still exists, however, this is not a real concern since the address space is not so scarce compared to the IPv4 world.

**Motivation/Benefits**: 🏗️ Lift restrictions, ✨ support more use cases/scenarios.

**Achievements:** VPN is configured to use IPv6 addresses only (even if the seed and shoot is IPv4-based). This lifts above mentioned restriction.

**Next Steps:** After `vpn2` has been [refactored to Golang](#⌨%EF%B8%8F-Rewrite-gardenervpn2-From-Bash-To-Golang), adapt the changes in the below linked draft PR, and make adaptations to support the high-availability case.

**Issue:** https://github.com/gardener/gardener/pull/9597#discussion_r1572358733

**Code/Pull Requests**: https://github.com/gardener/vpn2/pull/83

<hr />

## 👐 Harmonize Local VPN Setup With Real-World Scenario

**Problem Statement:**  In the local development setup, the VPN tunnel check performed by `gardenlet` (port-forward check) does not detect a broken VPN tunnel, because either `kube-apiserver` (HA clusters) or `vpn-seed-server` (non-HA clusters) route requests to the `kubelet` API directly via the seed's pod network. When the VPN connection is broken, `kubectl port-forward` and `kubectl logs` continue to work, while `kubectl top no` (`APIServices`, `Webhooks`, etc.) is broken. We should strive towards resolving this discrepancy between the local setup and real-world scenarios regarding the VPN connection to prevent bugs by validating the real setup in e2e tests.

**Motivation/Benefits**: 🧪 Increased output qualification.

**Achievements:** `provider-local` has been augmented to dynamically create Calico's `IPPool` resources. These are used for allocating IP addresses for the shoot worker pods according to the specified node CIDR in `.spec.networking.nodes`. This way, the VPN components are configured correctly to route traffic from control plane components to shoot kubelets via the tunnel. This aligns the local scenario with the real-world situation.

**Next Steps:** Review and merge the opened pull request.

**Issue:** https://github.com/gardener/gardener/issues/9604

**Code/Pull Requests**: https://github.com/gardener/machine-controller-manager-provider-local/pull/42, https://github.com/gardener/gardener/pull/9752

<hr />

## 🐝 Support Cilium `v1.15+` For HA `Shoot`s

**Problem Statement:** Cilium `v1.15` does not consider [`StatefulSet` labels in `NetworkPolicy`s](https://docs.cilium.io/en/v1.15/operations/performance/scalability/identity-relevant-labels/). Unfortunately, Gardener uses `statefulset.kubernetes.io/pod-name` in `Service`s/`NetworkPolicy`s to address individual `vpn-seed-server` pods for highly-available `Shoot`s. Therefore, the VPN tunnel does not work for such `Shoot`s in case the seed cluster runs Cilium `v1.15` or higher.

**Motivation/Benefits**: 🏗️ Lift restrictions, ✨ support more use cases/scenarios.

**Achievements:** A prototype has been developed making the `Service`s for the `vpn-seed-server` [headless](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services). Thereby, it is no longer required that the `StatefulSet` labels address individual `Pod`s. This simplifies the `NetworkPolicy`s and makes them work again with the mentioned Cilium release.

**Next Steps:** Decide on the final implementation of the solution within the Gardener networking team (there are alternate possible solutions/ideas).

**Code/Pull Requests**: https://github.com/ScheererJ/gardener/tree/enhancement/ha-vpn-headless

<hr />

## 🍞 Compression For `ManagedResource` `Secret`s

**Problem Statement:** The Kubernetes resources applied to garden, seed, or shoot clusters are stored in raw format (YAML) in `Secret`s within the garden runtime or seed cluster. These `Secret`s quickly and easily grow up in size, leading to considerable load for the ETCD cluster as well as network I/O (= costs) for basically all controllers watching `Secret`s.

**Motivation/Benefits**: 💰 Reduction of costs due to less traffic, 📈 better scalability due to less data volume.

**Achievements:** By leveraging the [Brotli compression algorithm](https://de.wikipedia.org/wiki/Brotli), we have been able to reduce the size of all `Secret`s by roughly a magnitude. This should have a great impact on network I/O and related costs.

**Next Steps:** Unit tests have to be written, and most unit tests for components deployed by `gardener/gardener` and extensions have to be adapted. The PR has to be opened.

**Code/Pull Requests**: https://github.com/metal-stack/gardener/tree/hackathon-mr-compression

<hr />

## 🚛 Making [Shoot Flux Extension](https://github.com/stackitcloud/gardener-extension-shoot-flux) Production-Ready

**Problem Statement:** [Continuing the track](https://github.com/gardener-community/hackathon/blob/main/2023-11_Schelklingen/README.md#-rework-shoot-flux-extension) of promoting the Flux extension to "production-ready" status, two main features are currently missing: Firstly, the Flux components were only installed once but never reconciled/updated anymore. Secondly, for some scenarios it might be necessary to provide additional `Secret`s (e.g., SOPS or encryption keys).

**Motivation/Benefits**: 🏗️ Lift restrictions, ✨ support more use cases/scenarios.

**Achievements:** A new `syncMode` field has been added to the extension's provider config API which can be used to control the reconciliation behaviour. In addition, the additional `Secret` resources are now synced into the cluster such that Flux can use them to decrypt/access resources.

**Next Steps:** Unit tests have to be implemented and the PR has to be opened. Generally, in order to release `v1.0.0`, the documentation and the `README.md` should be reworked.

**Code/Pull Requests**: https://github.com/stackitcloud/gardener-extension-shoot-flux/tree/sync-mode

<hr />

## 🧹 Move `machine-controller-manager-provider-local` Repository Into `gardener/gardener`

**Problem Statement:** The [`machine-controller-manager-provider-local`](https://github.com/gardener/machine-controller-manager-provider-local) implementation used by [`gardener-extension-provider-local`](https://github.com/gardener/gardener/blob/master/docs/extensions/provider-local.md) is currently maintained in a different GitHub repository. This makes related maintenance and development tasks more complicated and tedious.

**Motivation/Benefits**: 👨🏼‍💻 Improved developer productivity.

**Achievements:** The contents of the repository have been moved to `gardener/gardener`, and Skaffold has been augmented to dynamically build both the `node` image (used for local shoot worker node pods) and the `machine-controller-manager-provider-local` image (used for manging these worker node pods). Unfortunately, Skaffold does not support all features needed, so we had to introduce a few workarounds to make the e2e flow work. Still, the move will alleviate certain detours needed during development.

**Next Steps:** Once the opened PR got merged, the [`machine-controller-manager-provider-local`](https://github.com/gardener/machine-controller-manager-provider-local) repository should be archived/deleted.

**Code/Pull Requests**: https://github.com/gardener/gardener/pull/9782

<hr />

## 🗄️ Stop Vendoring Third-Party Code In OS Extensions

**Problem Statement:** Similar to [the last Hackathon's achievement](https://github.com/gardener-community/hackathon/blob/main/2023-11_Schelklingen/README.md#%EF%B8%8F-stop-vendoring-third-party-code-in-vendor-folder) regarding dropping the `vendor` folder in `gardener/gardener`, vendoring third-party code should also be avoided in the [OS extensions](https://github.com/gardener/gardener/blob/master/extensions/README.md#operating-system).

**Motivation/Benefits**: 👨🏼‍💻 Improved developer productivity by removing clutter from PRs and speeding up `git` operations.

**Achievements:** Two out of the four OS extensions maintained in the `gardener` GitHub organization have been adapted.

**Next Steps:** Replicate the work for [`os-ubuntu`](https://github.com/gardener/gardener-extension-os-ubuntu) and [`os-coreos`](https://github.com/gardener/gardener-extension-os-coreos).

**Code/Pull Requests**: https://github.com/gardener/gardener-extension-os-gardenlinux/pull/170, https://github.com/gardener/gardener-extension-os-suse-chost/pull/145

<hr />

## 📦 Consider Embedded Files For Local Image Builds 

**Problem Statement:** Currently, Gardener images are not automatically rebuilt by Skaffold for local development in case a file embedded into the code using [Golang's `embed` feature](https://pkg.go.dev/embed) changes. The reason is that we use `go list` to compute all package dependencies of the Gardener binaries, but `go list` cannot detect embedded files. Hence, they are not part of the dependency lists in the `skaffold.yaml` for the Gardener binaries. This makes the development process of these files tedious.

**Motivation/Benefits**: 👨🏼‍💻 Improved developer productivity.

**Achievements:** The [`hack` script](https://github.com/gardener/gardener/blob/master/hack/check-skaffold-deps-for-binary.sh) ([an achievement of a previous Hackathon](https://github.com/gardener-community/hackathon/blob/main/2023-11_Schelklingen/README.md#-auto-update-skaffold-dependencies)) computing the binary dependencies has been augmented to detect the embedded files and make them part of the list of dependencies.

**Next Steps:** Review and merge the pull request.

**Code/Pull Requests**: https://github.com/gardener/gardener/pull/9778

<hr />

![ApeiroRA](https://apeirora.eu/assets/img/BMWK-EU.png)
