# Kagent Architecture Guide

A comprehensive guide to understanding Kagent - the CNCF Sandbox Open Source AI Agent Framework for Kubernetes and SRE automation.

---

## What is Kagent?

Kagent is a **Kubernetes-native AI agent framework** created by Solo.io and now a **CNCF Sandbox project**. It enables DevOps and platform engineers to build, deploy, and run AI-powered solutions that automate complex operations and troubleshooting tasks in Kubernetes environments.

### Key Value Proposition

```
┌─────────────────────────────────────────────────────────────────┐
│                    BEFORE KAGENT                                 │
│  DevOps Engineer manually runs kubectl, helm, istioctl...       │
│  Time-consuming, error-prone, requires expertise                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    WITH KAGENT                                   │
│  "Scale deployment nginx to 5 replicas"                         │
│  AI Agent understands → selects tool → executes → reports       │
│  Automated, intelligent, accessible to all skill levels         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Open Standards Foundation

Kagent is built on open standards ensuring vendor independence:

| Standard | Purpose |
|----------|---------|
| **MCP (Model Context Protocol)** | Connects LLMs to external tools |
| **A2A (Agent-to-Agent Protocol)** | Enables multi-agent collaboration |
| **ADK (Agent Development Kit)** | Framework for building agents |

---

## Core Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        KAGENT ARCHITECTURE                           │
│                                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐  ┌───────────┐  │
│  │   KAGENT    │  │   KAGENT     │  │   KAGENT    │  │  KAGENT   │  │
│  │     UI      │  │  CONTROLLER  │  │   ENGINE    │  │    CLI    │  │
│  │ (Dashboard) │  │  (Watches    │  │  (Executes  │  │ (Command  │  │
│  │             │  │   CRDs)      │  │   Agents)   │  │   Line)   │  │
│  └──────┬──────┘  └──────┬───────┘  └──────┬──────┘  └───────────┘  │
│         │                │                 │                         │
│         └────────────────┼─────────────────┘                         │
│                          │                                           │
│                          ▼                                           │
│         ┌────────────────────────────────────┐                      │
│         │     KUBERNETES CUSTOM RESOURCES     │                      │
│         │  Agent | ModelConfig | ToolServer  │                      │
│         └────────────────────────────────────┘                      │
│                          │                                           │
│                          ▼                                           │
│         ┌────────────────────────────────────┐                      │
│         │          KAGENT TOOLS              │                      │
│         │         (MCP Server)               │                      │
│         │   K8s | Helm | Istio | Argo | etc  │                      │
│         └────────────────────────────────────┘                      │
│                          │                                           │
│                          ▼                                           │
│         ┌────────────────────────────────────┐                      │
│         │       KUBERNETES CLUSTER           │                      │
│         └────────────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Kagent Controller

The brain that manages agent lifecycle through Kubernetes custom resources.

```
┌─────────────────────────────────────────────────┐
│              KAGENT CONTROLLER                   │
│                                                  │
│  ┌──────────────┐    ┌──────────────────────┐   │
│  │    Watch     │    │   Reconcile Loop     │   │
│  │    CRDs      │ →  │   - Agent            │   │
│  │              │    │   - ModelConfig      │   │
│  │              │    │   - RemoteMCPServer  │   │
│  └──────────────┘    └──────────────────────┘   │
│                                                  │
│  ServiceAccount: kagent-controller               │
└─────────────────────────────────────────────────┘
```

**Responsibilities:**
- Watches Kagent custom resources
- Creates runtime resources (deployments, services)
- Manages agent-to-MCP server connections
- Handles LLM configuration

---

### 2. Kagent Engine (App)

Executes agent logic using the Agent Development Kit (ADK).

```
┌─────────────────────────────────────────────────┐
│                KAGENT ENGINE                     │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │           Agent Execution                 │   │
│  │                                           │   │
│  │  User Request                             │   │
│  │       ↓                                   │   │
│  │  System Prompt + LLM                      │   │
│  │       ↓                                   │   │
│  │  Tool Selection (via MCP)                 │   │
│  │       ↓                                   │   │
│  │  Execute & Return Response                │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

---

### 3. Kagent Tools (MCP Server)

The worker that executes actual Kubernetes/Helm/Istio commands.

```
┌─────────────────────────────────────────────────┐
│              KAGENT TOOLS (MCP SERVER)           │
│                                                  │
│  ┌─────────────────────────────────────────┐    │
│  │          Available Tool Categories       │    │
│  │  ┌─────────┐ ┌─────────┐ ┌───────────┐  │    │
│  │  │   K8s   │ │  Helm   │ │   Istio   │  │    │
│  │  │  Tools  │ │  Tools  │ │   Tools   │  │    │
│  │  └─────────┘ └─────────┘ └───────────┘  │    │
│  │  ┌─────────┐ ┌─────────┐ ┌───────────┐  │    │
│  │  │  Argo   │ │ Cilium  │ │Prometheus │  │    │
│  │  │  Tools  │ │  Tools  │ │   Tools   │  │    │
│  │  └─────────┘ └─────────┘ └───────────┘  │    │
│  └─────────────────────────────────────────┘    │
│                                                  │
│  ServiceAccount: kagent-tools                    │
│  ⚠️ NEEDS RBAC PERMISSIONS TO EXECUTE COMMANDS  │
│                                                  │
│  Endpoint: http://kagent-tools.kagent:8084/mcp  │
└─────────────────────────────────────────────────┘
```

**Key Point:** This component needs RBAC permissions because it actually runs kubectl/helm commands!

---

### 4. Kagent UI (Dashboard)

Web interface for managing agents and conversations.

| Feature | Description |
|---------|-------------|
| Chat Interface | Natural language conversation with agents |
| Agent Management | View and configure agents |
| Session History | Browse past conversations |
| Tool Monitoring | See which tools are being used |

---

### 5. Kagent CLI

Command-line interface for operations.

```bash
# Example CLI commands
kagent agent list
kagent agent create my-agent
kagent tool list
```

---

## KMCP - MCP Kubernetes Toolkit

KMCP enables developers to create, deploy, and securely run MCP servers on Kubernetes.

### What is MCP (Model Context Protocol)?

MCP is a protocol that connects LLMs to external tools and data sources.

```
┌─────────────┐         MCP Protocol          ┌─────────────────┐
│             │  ←─────────────────────────→  │                 │
│     LLM     │    Tool Calls & Responses     │   MCP Server    │
│   (Claude,  │                               │  (kagent-tools) │
│    GPT,     │    1. "What tools exist?"     │                 │
│   Ollama)   │    2. "Call k8s_get_pods"     │  Executes real  │
│             │    3. Returns pod list        │  kubectl cmds   │
└─────────────┘                               └─────────────────┘
```

### MCP Flow in Kagent

```
Step 1: Tool Discovery
┌─────────────────────────────────────────────────────────────┐
│  MCP Server exposes available tools:                         │
│  - k8s_get_resources                                         │
│  - k8s_create_resource                                       │
│  - helm_upgrade                                              │
│  - istio_analyze_cluster_configuration                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
Step 2: Agent Receives User Request
┌─────────────────────────────────────────────────────────────┐
│  User: "List all pods in namespace default"                  │
│  Agent (LLM): Analyzes request, selects k8s_get_resources   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
Step 3: MCP Tool Call
┌─────────────────────────────────────────────────────────────┐
│  Agent calls: k8s_get_resources(resource="pods", ns="default")│
│  MCP Server executes: kubectl get pods -n default            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
Step 4: Response
┌─────────────────────────────────────────────────────────────┐
│  MCP Server returns pod list                                 │
│  Agent formats response for user                             │
└─────────────────────────────────────────────────────────────┘
```

### Supported MCP Frameworks

| Framework | Language | Description |
|-----------|----------|-------------|
| FastMCP | Python | Quick MCP server development |
| MCP Go | Go | Go-based MCP servers |

---

## Kubernetes Custom Resources (CRDs)

Kagent uses CRDs to define all components declaratively.

### Agent CRD

Defines an AI agent with its capabilities.

```yaml
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: k8s-agent
  namespace: kagent
spec:
  description: "Kubernetes Expert AI Agent"
  declarative:
    modelConfig: default-model-config      # Which LLM to use
    systemMessage: |                       # Agent instructions
      You are a Kubernetes expert...
    tools:                                 # Available tools
    - mcpServer:
        name: kagent-tool-server
        toolNames:
          - k8s_get_resources
          - k8s_create_resource
          - helm_upgrade
```

**Agent Components:**
| Component | Purpose |
|-----------|---------|
| Instructions | System prompt defining behavior |
| Tools | Executable capabilities (via MCP) |
| Skills | Behavioral guidance for tool usage |

**Note:** Agents can use other agents as tools (hierarchical automation)!

---

### RemoteMCPServer CRD

Defines connection to an MCP tool server.

```yaml
apiVersion: kagent.dev/v1alpha2
kind: RemoteMCPServer
metadata:
  name: kagent-tool-server
  namespace: kagent
spec:
  url: http://kagent-tools.kagent:8084/mcp
  protocol: STREAMABLE_HTTP
  timeout: 30s
status:
  discoveredTools:      # Auto-discovered from server
    - k8s_get_resources
    - k8s_create_resource
    - helm_upgrade
    # ... more tools
```

---

### ModelConfig CRD

Defines LLM provider configuration.

```yaml
apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: default-model-config
  namespace: kagent
spec:
  provider: openai          # or anthropic, ollama, azure, vertex
  model: gpt-4
  apiKey:
    secretRef:
      name: llm-api-key
      key: api-key
```

---

## Supported LLM Providers

| Provider | Configuration |
|----------|---------------|
| OpenAI | API key |
| Azure OpenAI | Azure credentials |
| Anthropic (Claude) | API key |
| Google Vertex AI | GCP credentials |
| Google Gemini | API key |
| Amazon Bedrock | AWS credentials |
| Ollama | Local endpoint URL |
| Custom (AI Gateway) | Compatible endpoint |

---

## Complete Tool Categories

### Kubernetes Tools (k8s_*)

| Tool | Description | Mode |
|------|-------------|------|
| `k8s_get_resources` | List resources (pods, deployments, etc.) | Read |
| `k8s_get_available_api_resources` | List API resources | Read |
| `k8s_describe_resource` | Detailed resource info | Read |
| `k8s_get_resource_yaml` | Get YAML of resource | Read |
| `k8s_get_pod_logs` | View pod logs | Read |
| `k8s_get_events` | Cluster events | Read |
| `k8s_get_cluster_configuration` | Cluster config | Read |
| `k8s_check_service_connectivity` | Test service connection | Read |
| `k8s_create_resource` | Create new resource | Write |
| `k8s_create_resource_from_url` | Create from URL | Write |
| `k8s_apply_manifest` | Apply YAML manifest | Write |
| `k8s_patch_resource` | Update resource | Write |
| `k8s_delete_resource` | Delete resource | Write |
| `k8s_label_resource` | Add labels | Write |
| `k8s_remove_label` | Remove labels | Write |
| `k8s_annotate_resource` | Add annotations | Write |
| `k8s_remove_annotation` | Remove annotations | Write |
| `k8s_scale` | Scale deployments | Write |
| `k8s_rollout` | Manage rollouts | Write |
| `k8s_execute_command` | Exec into pods | Write |
| `k8s_generate_resource` | Generate YAML | Read |

### Helm Tools (helm_*)

| Tool | Description |
|------|-------------|
| `helm_upgrade` | Install or upgrade a release |
| `helm_uninstall` | Remove a release |
| `helm_list_releases` | List all releases |
| `helm_get_release` | Get release details |
| `helm_repo_add` | Add Helm repository |
| `helm_repo_update` | Update repositories |

### Istio Tools (istio_*)

| Tool | Description |
|------|-------------|
| `istio_install_istio` | Install Istio |
| `istio_analyze_cluster_configuration` | Analyze for issues |
| `istio_version` | Get version info |
| `istio_proxy_status` | Envoy proxy status |
| `istio_proxy_config` | Proxy configuration |

### Argo Tools (argo_*)

| Tool | Description |
|------|-------------|
| `argo_rollouts_list` | List rollouts |
| `argo_pause_rollout` | Pause rollout |
| `argo_promote_rollout` | Promote rollout |
| `argo_set_rollout_image` | Set image |

### Cilium Tools (cilium_*)

| Tool | Description |
|------|-------------|
| `cilium_install_cilium` | Install Cilium |
| `cilium_status_and_version` | Status check |
| `cilium_get_endpoints_list` | List endpoints |
| `cilium_list_services` | List services |

### Prometheus Tools (prometheus_*)

| Tool | Description |
|------|-------------|
| `prometheus_query_tool` | Execute PromQL query |
| `prometheus_query_range_tool` | Range query |
| `prometheus_promql_tool` | Generate PromQL |
| `prometheus_targets_tool` | Get targets |

---

## RBAC Architecture

### Why Two ServiceAccounts Need Permissions

```
┌────────────────────────────────────────────────────────────────┐
│                    PERMISSION FLOW                              │
│                                                                 │
│  User Request: "Delete pod nginx-123"                          │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────────┐                                          │
│  │ kagent-controller│  Manages agent resources                 │
│  │ (SA: kagent-     │  Needs: create/update deployments        │
│  │  controller)     │                                          │
│  └────────┬─────────┘                                          │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                          │
│  │  kagent-tools    │  ACTUALLY RUNS kubectl delete pod        │
│  │  (SA: kagent-    │  Needs: delete pods (and everything      │
│  │   tools)         │         else you want to do!)            │
│  └────────┬─────────┘                                          │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                          │
│  │ Kubernetes API   │                                          │
│  └──────────────────┘                                          │
└────────────────────────────────────────────────────────────────┘
```

### Required ClusterRoleBindings

```yaml
# 1. For kagent-controller
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kagent-controller-admin-binding
subjects:
- kind: ServiceAccount
  name: kagent-controller
  namespace: kagent
roleRef:
  kind: ClusterRole
  name: kagent-admin-role
  apiGroup: rbac.authorization.k8s.io

---
# 2. For kagent-tools (CRITICAL!)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kagent-tools-admin-binding
subjects:
- kind: ServiceAccount
  name: kagent-tools
  namespace: kagent
roleRef:
  kind: ClusterRole
  name: kagent-admin-role
  apiGroup: rbac.authorization.k8s.io
```

---

## Agent-to-Agent (A2A) Communication

Kagent supports multi-agent collaboration:

```
┌─────────────────────────────────────────────────────────────┐
│                   A2A ARCHITECTURE                           │
│                                                              │
│  ┌─────────────────┐                                        │
│  │  Primary Agent  │                                        │
│  │  (Orchestrator) │                                        │
│  └────────┬────────┘                                        │
│           │                                                  │
│     ┌─────┴─────┬─────────────┐                             │
│     ▼           ▼             ▼                             │
│  ┌──────┐   ┌──────┐    ┌──────────┐                       │
│  │ K8s  │   │ Helm │    │ Security │                       │
│  │Agent │   │Agent │    │  Agent   │                       │
│  └──────┘   └──────┘    └──────────┘                       │
│                                                              │
│  Agents can delegate tasks to specialized sub-agents        │
└─────────────────────────────────────────────────────────────┘
```

---

## Observability

Kagent supports OpenTelemetry tracing for monitoring agents and tools.

```
┌─────────────────────────────────────────────────────────────┐
│                   OBSERVABILITY                              │
│                                                              │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────────┐    │
│  │ Kagent  │───▶│ OpenTelemetry│───▶│ Jaeger/Zipkin   │    │
│  │ Engine  │    │   Collector  │    │ Grafana/etc     │    │
│  └─────────┘    └──────────────┘    └─────────────────┘    │
│                                                              │
│  Traces include:                                             │
│  - Agent execution time                                      │
│  - Tool calls and responses                                  │
│  - LLM API latency                                           │
│  - Error tracking                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Problem-Solving Capabilities

Kagent helps with these operational challenges:

| Challenge | How Kagent Helps |
|-----------|------------------|
| Complex DevOps workflows | Automate multi-step operations |
| Multi-hop connection failures | Diagnose network issues |
| Application unreachability | Debug service connectivity |
| Performance degradation | Analyze metrics, suggest fixes |
| Gateway/HTTPRoute issues | Troubleshoot traffic management |

---

## Deployment Architecture

```
Namespace: kagent
│
├── Deployments
│   ├── kagent-controller (SA: kagent-controller)
│   ├── kagent-tools (SA: kagent-tools)
│   └── kagent-ui (optional)
│
├── Services
│   ├── kagent-tools (port 8084 - MCP endpoint)
│   └── kagent-ui (port 80/443)
│
├── Custom Resources
│   ├── Agent: k8s-agent
│   ├── RemoteMCPServer: kagent-tool-server
│   └── ModelConfig: default-model-config
│
├── Secrets
│   └── llm-api-key (LLM provider credentials)
│
└── RBAC
    ├── ClusterRole: kagent-admin-role
    ├── ClusterRoleBinding: kagent-controller-admin-binding
    └── ClusterRoleBinding: kagent-tools-admin-binding
```

---

## Complete Workflow Example

### User: "Create a deployment named nginx with 3 replicas"

```
Step 1: User Input
┌─────────────────────────────────────────────────────────────┐
│ User types: "Create a deployment named nginx with 3 replicas"│
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ▼
Step 2: Kagent UI → Agent
┌─────────────────────────────────────────────────────────────┐
│ UI sends request to k8s-agent                                │
│ Agent pod receives the request                               │
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ▼
Step 3: LLM Processing
┌─────────────────────────────────────────────────────────────┐
│ System Prompt: "You are a Kubernetes expert..."              │
│ User Message: "Create a deployment named nginx..."           │
│                                                              │
│ LLM Response: Use k8s_create_resource tool with:            │
│ - kind: Deployment                                           │
│ - name: nginx                                                │
│ - replicas: 3                                                │
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ▼
Step 4: MCP Tool Call
┌─────────────────────────────────────────────────────────────┐
│ Agent calls RemoteMCPServer (kagent-tool-server)            │
│ Tool: k8s_create_resource                                    │
│ Parameters: deployment YAML with nginx, 3 replicas          │
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ▼
Step 5: Tool Execution
┌─────────────────────────────────────────────────────────────┐
│ kagent-tools (using SA: kagent-tools)                       │
│ Executes: kubectl create deployment nginx --replicas=3      │
│ Uses RBAC permissions to create resource                    │
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ▼
Step 6: Response
┌─────────────────────────────────────────────────────────────┐
│ Tool returns: "deployment.apps/nginx created"               │
│ Agent formats: "I've created the nginx deployment with      │
│                 3 replicas. You can verify with..."         │
│ UI displays response to user                                │
└─────────────────────────────────────────────────────────────┘
```

---

## Security Modes

### Read-Only Mode (Default)

```yaml
toolNames:
  - k8s_get_resources
  - k8s_describe_resource
  - k8s_get_pod_logs
  # Only read operations
```

- Safe for production monitoring
- No risk of accidental changes
- Good for initial deployment

### Full Mutation Mode

```yaml
toolNames:
  - k8s_get_resources
  - k8s_create_resource
  - k8s_delete_resource
  - k8s_patch_resource
  # Read + Write operations
```

- Requires RBAC setup for kagent-tools
- Use in lab/dev environments
- Full automation capability

---

## Troubleshooting Commands

```bash
# Check all Kagent components
kubectl get all -n kagent

# View CRDs
kubectl get crd | grep kagent

# Check agent configuration
kubectl get agent -n kagent -o yaml

# Check available tools from MCP server
kubectl get remotemcpserver -n kagent -o yaml

# View controller logs
kubectl logs deployment/kagent-controller -n kagent

# View tool server logs
kubectl logs deployment/kagent-tools -n kagent

# Check RBAC permissions
kubectl auth can-i create pods --as=system:serviceaccount:kagent:kagent-tools
kubectl auth can-i delete deployments --as=system:serviceaccount:kagent:kagent-tools

# Restart components
kubectl rollout restart deployment kagent-controller -n kagent
kubectl rollout restart deployment kagent-tools -n kagent
```

---

## Notable Adopters

- Solo.io (Creator)
- Amdocs
- Au10tix
- Krateo

---

## Resources

| Resource | URL |
|----------|-----|
| Documentation | https://kagent.dev |
| GitHub | https://github.com/kagent-dev/kagent |
| CNCF Sandbox | Cloud Native Computing Foundation |

---

## Summary

| Component | Purpose | ServiceAccount |
|-----------|---------|----------------|
| Kagent UI | User interface | N/A |
| Kagent Controller | Manages CRDs, creates resources | kagent-controller |
| Kagent Engine | Executes agent logic (ADK) | - |
| Kagent Tools | Runs kubectl/helm commands (MCP) | kagent-tools |
| Kagent CLI | Command-line management | - |

**Key Insight**: The `kagent-tools` ServiceAccount needs RBAC permissions because it's the component that actually executes Kubernetes commands through the MCP protocol.

---

## Version Information

- Current Version: v0.7.18
- License: Apache 2.0
- Status: CNCF Sandbox Project
