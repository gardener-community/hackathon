# Hack The Garden 05/2023 Wrap Up

## 🕵️ Introduction Of `gardener-node-agent`

**Problem Statement:** `cloud-config-downloader` is an ever-growing shell script running on all worker nodes of all shoot clusters. It is templated via Golang and has a high complexity and development burden. It runs every `60s` and checks whether new systemd units or files have to be downloaded. There are several scalability drawbacks due to less flexibility with shell scripts compared to a controller-based implementation, for example unnecessary restarts of systemd units (e.g., `kubelet`) just because the shell script changed (which often results in short interrupts/hick-ups for end-users).

**Motivation/Benefits**: 💰 Reduction of costs due to less traffic, 📈 better scalability due to less API calls, ⛔️ prevented interrupts/hick-ups due to undesired `kubelet` restarts,  👨🏼‍💻 improved developer productivity, 🔧 reduced operation complexity

**Achievements:** A new Golang implementation for the `gardener-node-agent` based on `controller-runtime` has been developed next to the current `cloud-config-downloader` shell script. It gets deployed onto the nodes via an initial `gardener-node-init` script executed via user data which only downloads the `gardener-node-agent` binary from the OCI registry. From then on, only the `node-agent` runs as systemd unit and manages everything related to the node, including the `kubelet` and its own binaries (self-upgrade is possible).

**Next Steps:** Open an issue explaining the motivation about the change. Add missing integration tests, documentation, and reasonably split the changes into smaller, manageable, reviewable PRs. Add dependencies between config files and corresponding units such that only units get restarted when there are relevant changes.

**Code**: https://github.com/metal-stack/gardener/tree/gardener-node-agent, https://github.com/metal-stack/gardener/tree/gardener-node-agent-2

<hr />

## 🌐 IPv6 On Cloud Provider

**Problem Statement:** With [GEP-21](https://github.com/gardener/gardener/issues/7051), we have started to tackle IPv6 in the local setup. What needs to be done on cloud providers has not been covered or concretely evaluated so far, however this is obviously where eventually we have to go to.

**Motivation/Benefits**: 🔭 Future-proofness of Gardener, ✨ support more use-cases/scenarios

**Achievements:** With the help of an IPv6-only seed cluster provisioned via `kubeadm` on a GCP dev box, an IPv6-only shoot cluster could successfully be created. However, this required quite a lot of manual steps that were hard to automate for now. As expected, qualities on the various infrastructures differ quite a lot which increases the complexity of the entire topic.

**Next Steps:** The many manually performed steps must be documented. The changes already prepared by StackIT for the local setup in the scope of GEP-21 have to be merged (no open PR yet). The infrastructure providers have to be extended to incorporate the current learnings and to achieve an automated setup.
Generally, we have to discuss whether we really want to go to IPv6-only clusters. Instead, it is more likely that we require dual-stack clusters so that we don't have to setup separate landscapes just to provision IPv6 clusters. This comes with its own complexity and has not been tackled yet.

**Code:** https://github.com/stackitcloud/gardener/tree/hackathon-ipv6

<hr />

## 🌱 Bootstrapping "Masterful Clusters" aka "Autonomous Shoots"

**Problem Statement:** Gardener requires an initial cluster such that it can be deployed and spin up further seed or shoot clusters. This inital cluster can only be setup via different means than `Shoot` clusters, hence it has different qualities, hence it makes the landscapes more heterogenous, hence it eventually results in additional operational complexity.

**Motivation/Benefits**: 🔗 Drop third-party tooling dependency, 🔧 reduced operation complexity, ✨ support more use-cases/scenarios

**Achievements:** Gardener produces the needed static pod manifests for the control plane components (e.g., `kube-apiserver`) via existing code and puts them into `ConfigMap`s in the shoot namespace in the seed (eventually, the goal is to add them as units/files in the `OperatingSystemConfig`). Hence, an initial cluster for bootstrapping a first temporary seed and shoot would still be required. However, the idea is to drop it again after control plane has been moved into the created shoot cluster.

**Next Steps:** There are still many open questions and issues, i.e. how to apply the static pods for the control plane components automatically? How to get valid kubeconfigs for the scenario where `kube-apiserver` runs as static pod in the cluster itself? Generally, much more further brainstorming how to design the overall concept is required, and more people need to be involved. We should document all our current findings for future explorations.

**Code**: https://github.com/MartinWeindel/gardener/tree/initial-cluster-poc

<hr />

## 🔑 Garden Cluster Access For Extensions In Seed Clusters

**Problem Statement:** Extensions run in seed clusters and have no access to the garden cluster. In some scenarios, such access is required to cover certain use-cases. There are extension implementations that work around this limitation by re-using `gardenlet`'s kubeconfig for talking to the garden cluster. This is obviously bad since it prevents auditibility.

**Motivation/Benefits**: 🛡️ Increased security due to prevention of privilege escalation in the garden cluster by extensions, 👀 better auditing and transparency for garden access, 💡 basis for dropping technical debts (`extensions.gardener.cloud/v1alpha1.Cluster` resource), ✨ support more use-cases/scenarios

**Achievements:** The `tokenrequestor` controller part of `gardener-resource-manager` has been made reusable and is now also registered as part of `gardenlet`. This way, when `gardenlet`'s `ControllerInstallation` controller deploys extensions, it can automatically create dedicated "access token secrets" for the garden cluster. Then its `tokenrequestor` controller can easily provision them and maintain them in the extension namespaces in the seed cluster. All `Deployment`s and similar workload resources get automatically a volume mount for the garden kubeconfig and an environment variable pointing to the location of the kubeconfig. The RBAC privileges for all such extensions are at least similary restricted as for `gardenlet`, i.e. only resources related to the seed cluster they are responsible for can be accessed.

**Next Steps:** Open an proposal issue with a description about the changes. Technically, the changes are ready and only integration tests and documentation are missing, but let's wait for more feedback on the issue first.

**Code:** https://github.com/rfranzke/gardener/tree/hackathon/extensions-garden-access

<hr />

## 💾 Replacement Of `ShootState`s With Data In Backup Buckets

**Problem Statement:** The `ShootState`s are used to persist secrets and relevant extension data (infrastructure state/machine status, etc.) such that they can be restored into the new seed cluster during a shoot control plane migration. However, this resource results in quite a lot of frequent updates (hence network I/O and costs) on both the seed and garden cluster API servers since every change in the shoot worker nodes must be persisted (to prevent losing nodes during a migration). Besides, the approach has scalability concerns since the data does not really qualify for being stored in ETCD.

**Motivation/Benefits**: 💰 Reduction of costs due to less traffic, 📈 better scalability due to less API calls, 😅 simplified code and handling of the Shoot control plane migration process due to elimination of the `ShootState` resource and controllers

**Achievements:** Two new resources in the `extensions.gardener.cloud` API group were introduced: `BackupUpload` and `BackupDownload`. Corresponding controller implementations were added to the extensions library and to `provider-local`. The control plane migration process was adapted to use the new resources for uploading and downloading the relevant shoot state information to and from object storage buckets. A flaw in the `BackupEntry` API design related to the deletion of entries was discovered and an appropriate redesign was decided on.

**Next Steps:** Technically, tests and documentation are still needed. Also, we should regularly upload (every hour or so) a backup of `ShootState`. It probably makes sense to write up the proposal in a GEP and agree on the API changes.

**Code:** https://github.com/rfranzke/gardener/tree/hackathon/shootstate-s3

<hr />

## 🔐 Introducing `InternalSecret` Resource In Gardener API

**Problem Statement:** End-users can read or write `Secret`s in their project namespaces in the garden cluster. This prevents Gardener components from storing certain "Gardener-internal" secrets related to the `Shoot` in the respective project namespace (since end-users would have access to them). Also, the Gardener API server uses `ShootState`s for the `adminkubeconfig` subresource of `Shoot`s (needed for retrieving the client CA key used for signing short-lived client certificates).

**Motivation/Benefits**: ✨ Support more use-cases/scenarios, 😅 drop workarounds and dependencies on other resources in the garden cluster

**Achievements:** The Gardener `core.gardener.cloud` API gets a new `InternalSecret` resource which is very similar to the `core/v1.Secret`. The secrets manager has been made generic such that it can handle both `core/v1.Secret`s and `core.gardener.cloud/v1beta1.InternalSecret`s. `gardenlet` populates the client CA to an `InternalSecret` in the project namespace. This can be used by the API server to issue client certificates for the `adminkubeconfig` subresource and opens up for dropping the `ShootState` API.

**Next Steps:** Collect feedback in the proposal issue. Adapt missing tests and documentation, but otherwise implementation is ready.

**Issue**: [gardener/gardener#7999](https://github.com/gardener/gardener/issues/7999)

**Code**: https://github.com/rfranzke/gardener/tree/hackathon/internal-secrets

<hr />

## 🤖 Moving `machine-controller-manager` Deployment Responsibility To `gardenlet`

**Problem Statement:** The `machine-controller-manager` is deployed by all provider extensions even though it is always the same manifest/specifciation. The only provider-specific part is the sidecar container which implements the interface to the cloud provider API. This increases development efforts since changes have to replicated again and again for all extensions.

**Motivation/Benefits**: 👨🏼‍💻 Improved developer productivity, 🔧 reduced operation complexity

**Achievements:** `gardenlet` takes over deployment of the generic part of `machine-controller-manager` and provider extensions inject their specific sidecar containers via a wehook. The MCM version is managed centrally in `gardener/gardener`, i.e. provider extensions are now only responsible for their specific sidecar image.

**Next Steps:** Add missing tests, adopt documentation, and open pull request. All provider extensions have to be revendored and adapted after this PR has been merged and released.

**Code**: https://github.com/rfranzke/gardener/tree/hackathon/mcm

<hr />

## 🎯 Improved E2E Test Accuracy For Local Control Plane Migration

**Problem Statement:** Essential logic for migrating worker nodes during a shoot control plane migration was not tested/skipped in the local scenario which is the basis for e2e tests in Gardener. This effectively increased development effort and complexity.

**Motivation/Benefits**: 👨🏼‍💻 Improved developer productivity

**Achievements:** `provider-local` now makes use of the essential logic in the extensions library which increases confidence for changes. The related PR has been opened and got merged already.

**Next Steps:** None.

**Code**: https://github.com/gardener/gardener/pull/7981

<hr />

## 🏗️ Moving `apiserver-proxy-pod-mutator` Webhook Into `gardener-resource-manager`

**Problem Statement:** The `kube-apiserver` pods of shoot clusters were running a webhook server as sidecar container which was used to inject an environment variable into workload pods for preventing unnecessary network hops. While all this is still quite valid, the webhook server should not be ran as a sidecar container in the `kube-apiserver` pod as this violates best practices and comes which other downsides (e.g., when the webhook server needs to be scaled up or down, the entire `kube-apiserver` pod must be restarted).

**Motivation/Benefits**: 👨🏼‍💻 Improved developer productivity, ⛔️ prevented interrupts/hick-ups due to undesired `kube-apiserver` restarts

**Achievements:** The logic of the webhook server was moved into `gardener-resource-manager` which already serves other similar webhook handlers for workload pods in shoot clusters. Missing integration tests and documentation has been added. The PR has been opened and is now under review.

**Next Steps:** None.

**Code:** https://github.com/gardener/gardener/pull/7980

<hr />

## 🗝️ ETCD Encryption For Custom Resources

**Problem Statement:** Currently, only `Secret`s are encrypted [at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) in the ETCDs of shoots. However, users might want to configure more resources that shall be encrypted in ETCD since they might contain confidential data similar to `Secret`s. Also, when `gardener-operator` deploys the Gardener control plane, additional resources like `ControllerRegistration`s or `ShootState`s must be encrypted as well.

**Motivation/Benefits**: 🔏 Improve cluster security, ✨ support new use cases/scenarios, 🔁 reuse code for the Gardener control plane management via `gardener-operator`

**Achievements:** The `Shoot` and `Garden` APIs were augmented to allow configuration of to-be-encrypted resources. First preparations in the corresponding handling in code were taken and an approach for implementing the feature was decided on.

**Next Steps:** Continue the implementation as part of the `gardener-operator` story. The complex part is to orchestrate dynamically added or removed resources such that they either become encrypted or decrypted.

**Code**: https://github.com/rfranzke/gardener/tree/hackathon/etcd-encryption
