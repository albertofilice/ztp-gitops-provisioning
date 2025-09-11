# Provisioning Cluster with Gitops

## Introduction

Automation and standardization are fundamental for any company. Zero Touch Provisioning (ZTP) is one of the key solutions for deploying OpenShift on Bare Metal. In this blog, we analyze how ZTP revolutionizes the deployment and provisioning of OpenShift clusters, with particular attention to integration with OpenShift Virtualization.

## The Value of ZTP

### Day 0: Pre-Installation Automation

During the Day 0 phase, before the cluster installation, ZTP allows you to automate the preparation of the infrastructure through ClusterCurator and Ansible Automation Platform:

**This approach allows you to:**
- Automatically configure networking
- Prepare the bare metal infrastructure
- Verify prerequisites
- Perform pre-installation health checks

### Day 1: Automatic Deployment

ZTP transforms the OpenShift deployment process from a manual and error-prone operation into a fully automated process. The main advantages include:

- **Standardization**: All clusters are deployed with identical configurations
- **Repeatability**: The process can be replicated infinitely with consistent results
- **Speed**: Significant reduction in time-to-market for new clusters
- **Reliability**: Elimination of human errors in the installation process

### Day 2: Management and Scalability

The true power of ZTP is manifested in the Day 2 phase, where:

- **Scalability**: Automatic addition of new nodes to the cluster
- **Policy Management**: Centralized implementation of security and compliance policies
- **Consistency**: Maintenance of consistency across multiple clusters
- **Automation**: Automated management of updates and patches

## Policy Management with RHACM

The PolicyGenerator is a fundamental tool of Red Hat Advanced Cluster Management (RHACM) that allows you to define and manage policies declaratively. It works as a "policy generator" that converts YAML definitions into actual Kubernetes policies, simplifying the management of complex configurations across multiple clusters.

#### How the PolicyGenerator Works

The PolicyGenerator operates through three main components:

1. **PolicySets**: Group related policies and define where they should be applied through placement rules
2. **Placements**: Determine on which clusters the policies should be applied
3. **Policy Definitions**: Define individual policies and their configurations

Here is a basic example of how it is used:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  name: policies
policySets:
...
# array of policySets to define
policies:
...
# array of policies to define
```

### Recommended Policy Organization

The following folder structure represents a recommended approach for organizing policies, but it can be adapted based on the specific needs of the organization. This organization is designed to scale well as the number of policies and managed clusters increases and to improve the maintainability of the released code.

```bash
policies
├── Global                                  # Global configurations for all clusters
│   ├── base-config                        # Essential base configurations
│   │   ├── policy-chrony.yaml            # Time synchronization
│   │   ├── policy-custom-ca.yaml         # Custom CA certificates
│   │   ├── policy-kubelet.yaml           # Kubelet configuration
│   │   └── policy-ssh-key.yaml           # SSH key management
│   ├── day2-config                        # Post-installation configurations
│   │   ├── policy-alertmanager-customrule.yaml    # Alerting rules
│   │   ├── policy-file-integrity-operator.yaml    # File integrity monitoring
│   │   ├── policy-ingress-certificate.yaml        # Ingress certificates
│   │   └── policy-storagecluster.yaml    # Storage configuration
│   ├── hub-config                         # Hub-specific configurations
│   │   ├── policy-clusterlogging.yaml    # Centralized logging
│   │   ├── policy-enable-ceph-toolbox.yaml # Ceph tools
│   │   └── policy-storagecluster.yaml    # Storage configuration
│   ├── kustomization.yaml                 # Kustomize file
│   ├── policy-generator.yaml              # Policy generator
│   ├── registry                           # Registry configurations
│   │   └── policy-registry.yaml          # Internal registry policy
│   ├── secrets-config                     # Secret management
│   │   ├── policy-external-secrets.yaml  # Integration with external vaults
│   │   └── policy-secretstore.yaml       # Secret store configuration
│   ├── security-auth                      # Security and authentication
│   │   ├── policy-authentication.yaml    # Authentication methods
│   │   ├── policy-disable-self-provisioner.yaml  # Self-service limitations
│   │   ├── policy-htpasswd.yaml          # Htpasswd authentication
│   │   └── policy-remove-kubeadmin.yaml  # Remove default admin
│   ├── storage                            # Storage configurations
│   │   ├── enable-multipath.yaml         # Enable multipath
│   │   └── policy-storagecluster.yaml    # Storage configuration
│   ├── testing                            # Test policies
│   │   ├── check-cluster-operator.yaml   # Cluster operator check
│   │   └── virtualizations                    # Virtualization configurations
│   │       ├── policy-install-mtv.yaml       # Migration Toolkit for Virtualization
│   │       ├── policy-install-nmstate.yaml   # Network state management
│   │       └── policy-install-virtualization.yaml # OpenShift Virtualization
│   ├── CHANGELOG.md                           # Reference document for changes made
│   └── README.md                              # General documentation
```

This structure offers several advantages:

1. **Logical Organization**:
   - Clear separation between global and specific policies
   - Easy identification of policies by type and purpose
   - Support for multi-region organizations

2. **Simplified Maintenance**:
   - Targeted updates by category
   - More efficient review of changes
   - Reduced risk of conflicts

3. **Governance and Compliance**:
   - Policy tracking by region
   - Simplified configuration audit
   - Centralized management of security policies

Example of a policy-generator.yaml using this structure:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  name: policies
placementBindingDefaults:
  # Set an explicit placement binding name to use rather than rely on the default.
  name: "placementbinding"

policyDefaults:
  categories:
    - "CM Configuration Management"
  controls:
    - "CM-2 Baseline Configuration"
  standards:
    - "NIST SP 800-53"
  namespaceSelector:
    include: ["*"]
    exclude: []
  namespace: open-cluster-management-policies
  remediationAction: inform
  complianceType: "musthave"
  severity: low
  pruneObjectBehavior: "None"
  metadataComplianceType: "musthave"

policySets:
  - name: global
    description: "Global configuration for all managed clusters"
    placement:
      name: "global-placement"
      labelSelector:
        matchExpressions:
          - key: vendor
            operator: In
            values:
              - OpenShift

    policies:
      - configure-alert-manager
      - enable-registry
      - ingress-certificate
      - install-cert-manager
      - install-external-secrets
      - install-lvm
      - oauth-configuration
      - remove-kubeadmin
      - service-account-aap
      - ocp-tools
      - kubelet-config
      - check-cluster-operator
      - check-job-failed
      - file-integrity

  - name: hub
    description: "Dedicated configurations for the hub cluster"
    placement:
      name: "hub-placement"
      labelSelector:
        matchExpressions:
          - key: cluster-name
            operator: In
            values:
              - hub
    policies:
      - automation-template
      - install-gitops

  - name: sno
    description: "Dedicated configurations for managed clusters of type sno"
    placement:
      name: "sno-placement"
      labelSelector:
        matchExpressions:
          - key: cluster-name
            operator: NotIn
            values:
              - sno
    policies:
      - install-smb-operator

policies:
  - name: configure-alert-manager
    description: "Configures AlertManager for notification management"
    manifests:
      - path: ../policies/alert-manager.yml
    remediationAction: enforce
    categories:
      - "CM Configuration Management"
    controls:
      - "CM-2 Baseline Configuration"
    standards:
      - "NIST SP 800-53"
    severity: low

  - name: install-external-secrets
    description: "Installs and configures External Secrets Operator for secret management"
    manifests:
      - path: ../policies/external-secrets.yml
    remediationAction: enforce
    categories:
      - "CM Configuration Management"
    controls:
      - "CM-2 Baseline Configuration"
    standards:
      - "NIST SP 800-53"
    severity: low
...
```

**This organization allows you to:**
- Maintain a clear separation of responsibilities
- Facilitate the addition of new policies
- Support multi-tenant management
- Simplify the review and approval process
- Improve traceability of changes

**Each folder can also contain a README.md that documents:**
- The purpose of the policies in that folder
- Dependencies with other policies
- Specific requirements for implementation
- Testing and validation procedures

Each organization should evaluate and adapt this structure based on its specific needs. The main factors to consider are:

- Number of managed clusters
- Policy complexity
- Compliance requirements
- Organizational team structure

## OpenShift Virtualization and ZTP

### Multi-Cluster Management

The integration between ZTP and OpenShift Virtualization brings significant benefits:

- **Provisioning**: Ease of installation and preparation of OpenShift clusters
- **Repeatability**: Easily replicate new installations always starting from a well-organized template structure.
- **Homogeneity**: ZTP and RHACM policies facilitate homogeneity and generalization of configurations applied to OpenShift clusters.

### Best Practices for Implementation

Before implementing ZTP and GitOps, it is important to follow a few essential best practices to ensure security, scalability, and ease of management. Here are the main recommendations:

1. **GitOps as the single source of truth**  
   All configurations should be versioned and managed through Git.
2. **Environment separation**  
   Keep separate environments (dev, test, prod) to avoid mistakes and simplify changes.
3. **Secure secret management**  
   Use operators like External Secrets to avoid storing sensitive data in the repository.
4. **Automated validation**  
   Integrate automated tests and compliance checks in CI/CD pipelines.
5. **Essential documentation**  
   Add brief documentation to configurations to support maintenance.

ArgoCD Application Example:

   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: policy
     namespace: openshift-gitops
   spec:
     destination:
       namespace: open-cluster-management-policies
       server: 'https://kubernetes.default.svc'
     project: default
     source:
       path: policies
       repoURL: 'git@github.com:albertofilice/rhacm-policies.git'
       targetRevision: HEAD
     syncPolicy:
       automated:
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
         - Validate=true
         - RespectIgnoreDifferences=true
   ```

## Automation with Ansible Automation Platform

Automating cluster lifecycle operations is essential for consistency and efficiency in large-scale environments. The Ansible Automation Platform enables the orchestration of complex workflows, such as pre- and post-installation tasks, upgrades, and validations, ensuring that every step is repeatable and auditable. Below are some key concepts and examples of how automation can be integrated into the ZTP process.

### Pre-hook and Post-hook

The ClusterCurator allows you to define pre and post installation or upgrade hooks:

```yaml
spec:
  install:
    towerAuthSecret: aap-integrations
    prehook:
      - name: Pre-Installation Check
        extra_vars:
          check_network: true
          check_storage: true
    posthook:
      - name: Post-Installation Validation
        extra_vars:
          validate_operators: true
          validate_network: true
  upgrade:
    towerAuthSecret: aap-integrations
    prehook:
      - name: Pre-Upgrade Check
        extra_vars:
          check_network: true
          check_storage: true
    posthook:
      - name: Post-Upgrade Validation
        extra_vars:
          validate_operators: true
          validate_network: true
        type: Job          
---
kind: Secret
apiVersion: v1
metadata:
  name: aap-integrations
  namespace: aap
data:
  host: <base64-encoded-host>
  token: <base64-encoded-token>
type: Opaque
```

## Key Benefits of the Git-Centric Approach

1. **Versioning and Control**
   - Complete tracking of changes
   - Possibility of rollback
   - Documentation embedded in the code

2. **Automation and CI/CD**
   - Automated deployment pipelines
   - Configuration testing
   - Pre-deployment validation

3. **Collaboration and Standardization**
   - Sharing of best practices
   - Structured review process
   - Facilitated knowledge sharing

## Day 1 Structure and Integration with ArgoCD

### Day 1 File Organization

The Day 1 phase represents the crucial moment of cluster installation. The file structure is organized to manage all aspects of the installation:

```bash
├── day1-agentclusterinstall.yaml      # Cluster installation configuration
├── day1-bmh.yaml                      # Bare Metal Host definition
├── day1-clusterdeployment.yaml        # Cluster deployment
├── day1-infraenv.yaml                 # Infrastructure environment
├── day1-klusterletaddonconfig.yaml    # Managed cluster addon configuration
├── day1-managed-cluster-secret.yaml   # Secret for the managed cluster
├── day1-managedcluster.yaml           # Managed cluster definition
├── day1-namespace.yaml                # Dedicated namespace
├── day1-nmstateconfig.yaml            # Network configuration
├── day1-oidc-configmap.yaml           # OIDC configuration
├── day1-pull-secret.yaml              # Pull secret for images
├── day1-reports.md                    # Documentation and reports
├── day2-workers/                      # Configurations for adding worker / Infra nodes
└── kustomization.yaml                 # Kustomize file for resource management
```

For management convenience, in this example we preferred to allow the management of worker nodes directly in the installed IPI cluster and not to have them managed by RHACM.

Therefore, in the Day 1 phase, a 3-node cluster will be configured and subsequently expanded through the dedicated ApplicationSet for cluster scaling.

### Automation with ArgoCD

The integration with ArgoCD allows you to automate the deployment process through ApplicationSet. Two main ApplicationSets manage the process:

1. **Environments ZTP**: Manages the initial deployment of clusters
2. **Cluster Scale**: Manages the scaling of worker nodes

#### ApplicationSet for Day 1

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: environments-ztp
  namespace: openshift-gitops
spec:
  generators:
    - git:
        directories:
          - path: '*'
        repoURL: 'https://github.com/test/ztp-environments.git'
        revision: develop
  # ... existing code ...
```

This ApplicationSet:
- Automatically scans directories in the Git repository
- Creates an application for each environment found
- Manages differences by ignoring specific fields to avoid conflicts
- Implements automated sync policies

#### ApplicationSet for Scaling

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-scale
  namespace: openshift-gitops
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            apps.open-cluster-management.io/acm-cluster: 'true'
  # ... existing code ...
```

This ApplicationSet:
- Selects managed clusters via label
- Applies scaling configurations from the day2-workers files
- Supports dynamic scaling of worker nodes

### Benefits of This Structure

1. **Complete Automation**:
   - Automated cluster deployment
   - Automatic scaling management
   - Self-healing of configurations

2. **Flexibility**:
   - Support for multiple environments
   - Easy addition of new clusters
   - Customizable configurations per environment

3. **Maintainability**:
   - Clear and organized structure
   - Separation of responsibilities
   - Integrated documentation

4. **Security**:
   - Centralized secret management
   - Access control via OIDC
   - Audit trail of changes

## Secret Management with External Secrets Operator

### Introduction

Within our ZTP project, we implemented External Secrets Operator (ESO) to securely and centrally manage all secrets necessary for cluster operation. This approach allows us to keep the Git repository clean from sensitive data, significantly improving the security of the infrastructure.

There are several certified products that allow you to integrate secrets via vault.

### Types of Secrets Managed

ESO manages various types of critical secrets for cluster operation:

1. **Access Credentials**:
   - Htpasswd credentials for authentication
   - Access tokens for registry
   - Credentials for Bare Metal Host (BMH)

2. **Certificates**:
   - Certificates for ingress
   - Custom CA certificates
   - Certificates for inter-cluster communication

3. **System Secrets**:
   - Pull secret for OpenShift
   - Secret for communication with RHACM
   - Token for integration with external services

### Implementation

Secret management is handled through two main components:

1. **SecretStore**: Defines the connection to the secret backend (e.g., HashiCorp Vault)
2. **ExternalSecret**: Defines how to map secrets from the backend to Kubernetes

Example configuration:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: open-cluster-management-policies
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso-role"
          serviceAccountRef:
            name: "eso-service-account"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: htpasswd-secret
  namespace: open-cluster-management-policies
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: htpasswd-secret
  data:
    - secretKey: htpasswd
      remoteRef:
        key: clusters/htpasswd
```

### Benefits of the Approach

1. **Security**:
   - No sensitive data in the Git repository
   - Automatic secret rotation
   - Centralized audit trail
   - Granular access control

2. **Maintainability**:
   - Centralized secret management
   - Automatic updates
   - Reduced risk of hardcoded secrets
   - Simplified rotation process

3. **Scalability**:
   - Support for multiple Vault instances
   - Efficient management of large quantities of secrets
   - Easy addition of new secrets
   - Support for multi-cluster environments

### Best Practices Implemented

1. **Environment Separation**:
   - Separate paths in Vault for each environment
   - Dedicated namespaces for secrets
   - Label for tracking secret origin

2. **Lifecycle Management**:
   - Automatic secret rotation
   - Secret versioning
   - Automatic backup of configurations

3. **Monitoring and Audit**:
   - Logging of secret operations
   - Alerting for unauthorized access
   - Periodic reports on usage

### Integration with the ZTP Workflow

ESO integrates perfectly with the ZTP workflow:

1. **Day 0**:
   - Prepare necessary secrets in Vault
   - Configure access policies
   - Setup SecretStore

2. **Day 1**:
   - Automated secret provisioning during installation
   - Setup initial credentials
   - Setup certificates

3. **Day 2**:
   - Automated secret rotation
   - Manage user credentials
   - Update certificates

### Security Considerations

When designing and implementing a secure and scalable OpenShift environment, it is important to consider several key aspects. The following points highlight the most critical considerations to keep in mind to ensure a robust and compliant infrastructure:

1. **Secret Access**:
   - Implement the principle of least privilege
   - Authentication based on service account
   - Audit logging of all accesses

2. **Encryption**:
   - Encrypted secrets at rest
   - TLS for communication with Vault
   - Secret key rotation

3. **Compliance**:
   - Compliance with security policies
   - Tracking of changes
   - Documentation of procedures

## Conclusion

The implementation of ZTP with a Git-centric approach represents a paradigm shift in the management of OpenShift infrastructure. The combination with OpenShift Virtualization allows you to create a robust, scalable, and easily manageable ecosystem, significantly reducing time-to-market and operational costs.

## References

- [Red Hat OpenShift Virtualization](https://www.redhat.com/en/technologies/cloud-computing/openshift/virtualization)
- [Red Hat Advanced Cluster Management](https://www.redhat.com/en/technologies/management/advanced-cluster-management)
- [Cluster Curator Controller](https://github.com/stolostron/cluster-curator-controller)
- [External Secrets Operator](https://external-secrets.io/latest/)
- [Repository Git](https://github.com/albertofilice/ztp-gitops-provisioning.git)
