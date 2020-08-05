# Azure Kubernetes Service (AKS) Secure Baseline Reference Implementation

This reference implementation demonstrates the _recommended starting (baseline) infrastructure architecture_ for an [AKS cluster](https://azure.microsoft.com/services/kubernetes-service). This is implementation and document is meant to guide an interdisciplinary team or multiple distinct teams like networking, security and development through the process of getting this secure baseline infrastructure deployed and understanding the components of it.

## Azure Architecture Center guidance

This project has a companion set of articles that describe challenges, design patterns, and best practices for a secure AKS cluster. You can find this article on the Azure Architecture Center at [Baseline architecture for a secure AKS cluster](https://aka.ms/architecture/aks-baseline). If you haven't reviewed it, we suggest you read it as it will give added context to the considerations applied in this implementation. Ultimately, this is the direct implementation of that specific architectural guidance.

## Architecture

**This architecture is infrastructure focused**, more so than workload. It concentrates on the AKS cluster itself, including concerns with identity, post-deployment configuration, secret management, and network topologies.

The implementation presented here is the _minimum recommended baseline for most AKS clusters_. This implementation integrates with Azure services that will deliver observability, provide a network topology that will support multi-regional growth, and keep the in-cluster traffic secure as well. This architecture should be considered your starting point for pre-production and production stages.

Throughout the reference implementation, you will see reference to _Contoso Bicycle_. They are a fictional small and fast-growing startup that provides online web services to its clientele on the west coast of North America. They have no on-premises data centers and all their containerized line of business applications are now about to be orchestrated by secure, enterprise-ready AKS clusters. You can read more about [their requirements and their IT team composition](./contoso-bicycle/README.md). This narrative provides grounding for some implementation details, naming conventions, etc. You should adapt as you see fit.

Finally, this implementation uses the [ASP.NET Core Docker sample web app](https://github.com/dotnet/dotnet-docker/tree/master/samples/aspnetapp) as an example workload. This workload purposefully uninteresting, as it is here exclusively to help you experience the baseline infrastructure.

### Core architecture components

#### Azure platform

* AKS v1.17
  * System and User [node pool separation](https://docs.microsoft.com/azure/aks/use-system-pools)
  * [AKS-managed Azure AD](https://docs.microsoft.com/azure/aks/managed-aad)
  * Managed Identities
  * Azure CNI
  * [Azure Monitor for containers](https://docs.microsoft.com/azure/azure-monitor/insights/container-insights-overview)
* Azure Virtual Networks (hub-spoke)
* Azure Application Gateway (WAF)
* AKS-managed Internal Load Balancers
* Azure Firewall

#### In-cluster OSS components

* [Flux GitOps Operator](https://fluxcd.io)
* [Traefik Ingress Controller](https://docs.microsoft.com/azure/dev-spaces/how-to/ingress-https-traefik)
* [Azure AD Pod Identity](https://github.com/Azure/aad-pod-identity)
* [Azure KeyVault Secret Store CSI Provider](https://github.com/Azure/secrets-store-csi-driver-provider-azure)
* [Kured](https://docs.microsoft.com/azure/aks/node-updates-kured)

![Network diagram depicting a hub-spoke network with two peered VNets, each with three subnets and main Azure resources.](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks/images/secure-baseline-architecture.svg)

## Deploy the reference implementation

A deployment of AKS-hosted workloads typically experiences a separation of duties and lifecycle management in the area of prerequisites, the host network, the cluster infrastructure, and finally the workload itself. This reference implementation is similar. Also, be aware our primary purpose is to illustrate the topology and decisions of a baseline cluster. We feel a "step-by-step" flow will help you learn the pieces of the solution and give you insight into the relationship between them. Ultimately, lifecycle/SDLC management of your cluster and its dependencies will depend on your situation (team roles, organizational standards, etc), and will be implemented as appropriate for your needs.

**Please start this learning journey in the _Preparing for the cluster_ section.** If you follow this through the end, you'll have our recommended baseline cluster installed, with an end-to-end sample workload running for you to reference in your own Azure subscription.

### 1. :rocket: Preparing for the cluster

There are considerations that must be addressed before you start deploying your cluster. Do I have enough permissions in my subscription and AD tenant to do a deployment of this size? How much of this will be handled by my team directly vs having another team be responsible?

* [ ] Begin by ensuring you [install and meet the prerequisites](./01-prerequisites.md)
* [ ] [Procure client-facing and AKS Ingress Controller TLS certificates](./02-ca-certificates.md)
* [ ] [Plan your Azure Active Directory integration](./03-aad.md)

### 2. Build target network

Microsoft recommends AKS be deploy into a carefully planned network; sized appropriately for your needs and with proper network observability. Organizations typically favor a traditional hub-spoke model, which is reflected in this implementation. While this is a standard hub-spoke model, there are fundamental sizing and portioning considerations included that should be understood.

* [ ] [Build the hub-spoke network](./04-networking.md)

### 3. Deploying the cluster

This is the heart of the guidance in this reference implementation; paired with prior network topology guidance. Here you will deploy the Azure resources for your cluster and the adjacent services such as Azure Application Gateway WAF, Azure Monitor, Azure Container Registry, and Azure Key Vault. This is also where you put the cluster under GitOps orchestration.

* [ ] [Deploy the AKS cluster and supporting services](./05-aks-cluster.md)
* [ ] [Place the cluster under GitOps management](./06-gitops.md)

We perform the prior steps manually here for you to understand the involved components, but we advocate for an automated DevOps process. Therefore, incorporate the prior steps into your CI/CD pipeline, as you would any infrastructure as code (IaC). We have included [a starter GitHub workflow](./github-workflow/aks-deploy.yaml) that demonstrates this.

### 4. Deploy your workload

Without a workload deployed to the cluster it will be hard to see how these decisions come together to work as a reliable application platform for your business. The deployment of this workload would typically follow a CI/CD pattern and may involve even more advanced deployment strategies (blue/green, etc). The following steps represent a manual deployment, suitable for illustration purposes of this infrastructure.

* [ ] Just like the cluster, there are [workload prerequisites to address](./07-workload-prerequisites.md)
* [ ] [Configure AKS Ingress Controller with Azure Key Vault integration](./08-secret-managment-and-ingress-controller.md)
* [ ] [Deploy the workload](./09-workload.md)

### 5. :checkered_flag: Validation

Now that the cluster and the sample workload is deployed; now it's time to look at how the cluster is functioning.

* [ ] [Perform end-to-end deployment validation](./10-validation.md)

## :broom: Clean up resources

Most of the Azure resources deployed in the prior steps will incur ongoing charges unless removed.

* [ ] [Cleanup all resources](./11-cleanup.md)

## Inner-loop development scripts

We have provided some sample deployment scripts that you could adapt for your own purposes while doing a POC/spike on this. Those scripts are found in the [inner-loop-scripts directory](./inner-loop-scripts). They include some additional considerations and may include some additional narrative as well. Consider checking them out. They consolidate most of the walk-through performed above into combined execution steps.

## Preview features

While this reference implementation tends to avoid _preview_ features of AKS to ensure you have the best customer support experience; there are some features you may wish to evaluate in pre-production clusters that augment your posture around security, manageability, etc. Consider trying out and providing feedback on the following. As these features come out of preview, this reference implementation may be updated to incorporate them.

* [Azure RBAC for Kubernetes Authentication](https://docs.microsoft.com/azure/aks/manage-azure-rbac) - An extension of the Azure AD integration already in this reference implementation. Allowing you to bind Kubernetes authentication to Azure RBAC role assignments.
* [Host-based encryption](https://docs.microsoft.com/azure/aks/enable-host-encryption) - Leverages added data encryption on your VMs' temp and OS disks.
* [Node image upgrades](https://docs.microsoft.com/azure/aks/node-image-upgrade) - Built-in OS patch management; replacement for BYO solutions, such as Kured used in this reference implementation.
* [AKS Ubuntu 18.04](https://docs.microsoft.com/azure/aks/cluster-configuration#os-configuration-preview) - Will be the default starting in AKS 1.18.
* [Containerd support](https://docs.microsoft.com/azure/aks/cluster-configuration#container-runtime-configuration-preview) - Replaces the use of Moby with Containerd directly, reducing node resource consumption and improving startup latency.
* [Generation 2 VM support](https://docs.microsoft.com/azure/aks/cluster-configuration#generation-2-virtual-machines-preview) - Increased memory options, Intel SGX support, and UEFI-based boot architectures.

## Advanced topics

This reference implementation intentionally does not cover more advanced scenarios. For example topics like the following are not addressed:

* Cluster lifecycle management with regard to SDLC and GitOps
* Workload SDLC integration (including concepts like [DevSpaces](https://docs.microsoft.com/azure/dev-spaces/), advanced deployment techniques, etc)
* Mapping decisions to [CIS benchmark controls](https://www.cisecurity.org/benchmark/kubernetes/)
* Container security
* Multi-region clusters
* [Advanced regulatory compliance](https://github.com/Azure/sg-aks-workshop) (FinServ)
* Multiple (related or unrelated) workloads owned by the same team
* Multiple workloads owned by disparate teams (AKS as a shared platform in your organization)
* Cluster-contained state (PVC, etc)
* Windows node pools
* Scale-to-zero node pools and event-based scaling (KEDA)
* [Private Kubernetes API Server](https://docs.microsoft.com/azure/aks/private-clusters)
* [Terraform](https://docs.microsoft.com/azure/developer/terraform/create-k8s-cluster-with-tf-and-aks)
* [Bedrock](https://github.com/microsoft/bedrock)
* [dapr](https://github.com/dapr/dapr)

Keep watching this space, as we build out reference implementation guidance on topics such as these. Further guidance delivered will use this baseline AKS implementation as their starting point. If you would like to contribute or suggest a pattern built on this baseline, [please get in touch](./CONTRIBUTING.md).

## Final thoughts

Kubernetes is a very flexible platform, giving infrastructure and application operators many choices to achieve their business and technology objectives. At points along your journey, you will need to consider when to take dependencies on Azure platform features, OSS solutions, support channels, regulatory compliance, and operational processes. **We encourage this reference implementation to be the place for you to _start_ architectural conversations within your own team; adapting to your specific requirements, and ultimately delivering a solution that delights your customers.**

## Related documentation

* [Azure Kubernetes Service Documentation](https://docs.microsoft.com/azure/aks/)
* [Microsoft Azure Well-Architected Framework](https://docs.microsoft.com/azure/architecture/framework/)
* [Microservices architecture on AKS](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices)

## Contributions

Please see our [contributor guide](./CONTRIBUTING.md).

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact <opencode@microsoft.com> with any additional questions or comments.

With :heart: from Microsoft Patterns & Practices, [Azure Architecture Center](https://aka.ms/architecture).
