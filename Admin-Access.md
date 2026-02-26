# Granting Admin Access to Kagent in GKE (Full Mutation Mode)

This guide explains how to give Kagent full administrative access to your GKE/EKS cluster.

By default, Kagent agents are deployed in read-only mode for safety.

This guide enables:

- Create resources
- Patch resources
- Delete resources
- Apply manifests
- Helm operations
- Full Kubernetes automation

> **Warning**: Use this only in lab or controlled environments.

---

## Architecture Overview

```
User (Kagent UI)
        │
        ▼
┌─────────────────────┐
│  kagent-controller  │  ← Manages agent definitions
│  SA: kagent-controller
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   kagent-tools      │  ← Executes actual K8s commands (NEEDS ADMIN ACCESS!)
│  SA: kagent-tools   │
└─────────┬───────────┘
          │
          ▼
   Kubernetes API Server
          │
          ▼
   Cluster Resources
```

**Important**: You must grant admin access to BOTH ServiceAccounts:
- `kagent-controller` - manages agents
- `kagent-tools` - executes the actual kubectl/helm commands

---

## STEP 1 — Verify ServiceAccounts

Check which ServiceAccounts Kagent uses:

```bash
kubectl get sa -n kagent
```

You should see:
- `kagent-controller`
- `kagent-tools`

---

## STEP 2 — Create Cluster Admin Role

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kagent-admin-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF
```

---

## STEP 3 — Bind Role to BOTH ServiceAccounts

### 3a. Bind to kagent-controller

```bash
kubectl apply -f - <<EOF
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
EOF
```

### 3b. Bind to kagent-tools (THIS IS CRITICAL!)

```bash
kubectl apply -f - <<EOF
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
EOF
```

---

## STEP 4 — Verify RBAC Bindings

```bash
kubectl get clusterrolebinding | grep kagent
```

You should see:
```
kagent-controller-admin-binding
kagent-tools-admin-binding
```

---

## STEP 5 — Check Available Tools in RemoteMCPServer

First, see what tools are available:

```bash
kubectl get remotemcpserver kagent-tool-server -n kagent -o yaml | grep -A 200 "discoveredTools:"
```

Note the exact tool names (e.g., `helm_upgrade` NOT `helm_upgrade_release`).

---

## STEP 6 — Update Agent with Correct Tool Names

Edit the agent:

```bash
kubectl edit agent k8s-agent -n kagent
```

Update the `toolNames` section with the EXACT names from the RemoteMCPServer:

```yaml
spec:
  declarative:
    tools:
    - mcpServer:
        apiGroup: kagent.dev
        kind: RemoteMCPServer
        name: kagent-tool-server
        toolNames:
        # Kubernetes Tools
        - k8s_get_resources
        - k8s_get_available_api_resources
        - k8s_get_events
        - k8s_get_pod_logs
        - k8s_get_resource_yaml
        - k8s_get_cluster_configuration
        - k8s_describe_resource
        - k8s_check_service_connectivity
        - k8s_execute_command
        - k8s_create_resource
        - k8s_create_resource_from_url
        - k8s_apply_manifest
        - k8s_patch_resource
        - k8s_delete_resource
        - k8s_label_resource
        - k8s_remove_label
        - k8s_annotate_resource
        - k8s_remove_annotation
        - k8s_scale
        - k8s_rollout
        - k8s_generate_resource
        # Helm Tools (use EXACT names from discoveredTools)
        - helm_upgrade
        - helm_uninstall
        - helm_list_releases
        - helm_get_release
        - helm_repo_add
        - helm_repo_update
      type: McpServer
```

Save and exit.

---

## STEP 7 — Update System Message (Optional)

Inside the same agent file, you can update the `systemMessage`:

```yaml
systemMessage: |
  You are a Kubernetes DevOps automation assistant.

  You are allowed to:
  - Create Kubernetes resources
  - Patch existing resources
  - Delete resources
  - Apply YAML manifests
  - Install/Upgrade/Uninstall Helm releases

  Before performing destructive operations:
  - Explain the action
  - Describe impact
  - Then proceed

  Avoid modifying:
  - kube-system namespace
  - Core cluster components
```

---

## STEP 8 — Restart Deployments

```bash
kubectl rollout restart deployment kagent-tools -n kagent
kubectl rollout restart deployment kagent-controller -n kagent
```

---

## STEP 9 — Verify Everything

### Check deployments are running:

```bash
kubectl get pods -n kagent
```

### Check tool server logs for errors:

```bash
kubectl logs deployment/kagent-tools -n kagent --tail=50
```

### Check RBAC is correct:

```bash
kubectl auth can-i create deployments --as=system:serviceaccount:kagent:kagent-tools
kubectl auth can-i delete pods --as=system:serviceaccount:kagent:kagent-tools
```

Both should return `yes`.

---

## Common Mistakes

| Mistake | Solution |
|---------|----------|
| Only granting RBAC to `kagent-controller` | Grant RBAC to `kagent-tools` as well |
| Using wrong tool names (e.g., `helm_install_release`) | Check `discoveredTools` in RemoteMCPServer for exact names |
| Not restarting deployments after changes | Always restart both `kagent-tools` and `kagent-controller` |

---

## Troubleshooting

### Agent can only create namespaces, nothing else

This means `kagent-tools` ServiceAccount doesn't have admin permissions. Run Step 3b.

### Helm commands not working

Check the exact tool names in RemoteMCPServer:

```bash
kubectl get remotemcpserver kagent-tool-server -n kagent -o yaml | grep helm
```

Use those exact names in your Agent's `toolNames` list.

### Permission denied errors in logs

```bash
kubectl logs deployment/kagent-tools -n kagent | grep -i "forbidden\|denied"
```

If you see errors, verify the ClusterRoleBinding exists:

```bash
kubectl get clusterrolebinding kagent-tools-admin-binding -o yaml
```

---

## Quick Setup (All Commands)

Run all these commands in sequence for a fresh setup:

```bash
# Create ClusterRole
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kagent-admin-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

# Bind to kagent-controller
kubectl apply -f - <<EOF
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
EOF

# Bind to kagent-tools (CRITICAL!)
kubectl apply -f - <<EOF
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
EOF

# Restart deployments
kubectl rollout restart deployment kagent-tools -n kagent
kubectl rollout restart deployment kagent-controller -n kagent

# Verify
kubectl auth can-i create deployments --as=system:serviceaccount:kagent:kagent-tools
kubectl auth can-i delete pods --as=system:serviceaccount:kagent:kagent-tools
```

---

## Summary

The key insight is that Kagent has TWO components that need permissions:

1. **kagent-controller** - Manages agent definitions
2. **kagent-tools** - Actually executes Kubernetes commands

Both need admin RBAC permissions for full mutation mode to work.
