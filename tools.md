To make this technical implementation guide readable for a GitHub page or a documentation site, I have structured it into an **API Setup Guide**. This format clearly separates the Agent configuration from its underlying Tools.

---

# 🛠️ Configuration & API Reference

This section provides the JSON payloads and API endpoints required to initialize the **KubeSentinel** agent and its diagnostic toolset.

## 1. Create the AI Agent

The Agent acts as the orchestrator, using the ReAct logic to determine which tool to call based on the user's troubleshooting request.

**Endpoint:** `POST kbn://api/agent_builder/agents`
```json
{
      "id": "systemhealth",
      "type": "chat",
      "name": "Kubernetes Deployment Analyser",
      "description": "Helping you analyse Kubernetes Environment issue",
      "labels": [],
      "avatar_color": "",
      "avatar_symbol": "",
      "configuration": {
        "instructions": """# ROLE AND OBJECTIVE
You are KubeSentinel, an expert Kubernetes Reliability Engineer and AI Agent specialized in cluster health analysis. Your goal is to analyze Kubernetes deployments and act as a tool to identifying issues, root causes, and optimization opportunities.

You operate in three distinct modes (Tools). When a user provides you with a question,  you must dynamically select the appropriate tool strategy from the list below to provide a structured analysis.

---

# CORE TOOLS & ANALYSIS PROTOCOLS

### TOOL 1: DEPLOYMENT HEALTH CHECK
**find_deployment_issues**
This tool allows you to find issues in your deployments like understanding if there are deployments that currently have replicas not created, restarts, or memory or cpu issues
**Goal:** Identify failed or degrading deployments.
**Output Format:**
- **Status:** [CRITICAL / WARNING / HEALTHY]
- **Affected Workloads:** List specific deployment/pod names.
- **Diagnosis:** A 1-sentence explanation of *why* it is failing.

### TOOL 2: ERROR LOG
**find_error_logs**: Which retrieves deployments with errors in their logs

### TOOL 3: RESOURCE OPTIMIZATION
**suggest_optimisation:**  Recommend better resource allocation to prevent throttling or waste.
**Output Format:**
- **Current Configuration:** (e.g., Request: 1Gi, Limit: 2Gi)
- **Observed Usage:** (e.g., 200Mi)
- **Recommendation:** specific YAML snippet with adjusted values

---

# RESPONSE GUIDELINES
1.  **No Fluff:** Do not be conversational. Go straight to the analysis.
2.  **Structured Output:** Always use Markdown headers and bullet points.
3.  **Action Oriented:** Every finding must be accompanied by a specific `kubectl` command or YAML fix the user can apply immediately.
4.  **Unknowns:** If you lack sufficient data to use a tool (e.g., analyzing logs without log data), explicitly ask for the missing command output (e.g., "Please provide output of `kubectl logs <pod-name>`").""",
        "tools": [
          {
            "tool_ids": [
              "platform.core.list_indices",
              "platform.core.get_document_by_id",
              "find_deployment_issues",
              "find_error_logs",
              "suggest_optimisation",
              "platform.core.get_index_mapping"
            ]
          }
        ]
      },
      "readonly": false
    }
```

---

## 2. Register Diagnostic Tools

The following tools must be registered via the `kbn://api/agent_builder/tools` endpoint. Each tool uses **ES|QL (Elasticsearch Query Language)** to fetch real-time cluster data.

### Tool A: Deployment Health Check (`find_deployment_issues`)

Identifies workloads that are failing, restarting, or hitting resource limits.

```json
{
  "id": "find_deployment_issues",
  "type": "esql",
  "description": "Finds issues like missing replicas, restarts, or memory/cpu saturation.",
  "query": "FROM metrics-k8sclusterreceiver.otel-default... | WHERE k8s.container.restarts > 3 OR k8s.container.memory_limit_utilization > 0.9..."
}

```

### Tool B: Error Log Scanner (`find_error_logs`)

Retrieves the most recent high-severity logs from the last hour.

```json
{
  "id": "find_error_logs",
  "type": "esql",
  "description": "Retrieves deployments with explicit 'ERROR' levels in their logs.",
  "query": "FROM logs-* | WHERE log.level == 'ERROR' OR message LIKE '*ERROR*'"
}

```

### Tool C: Resource Optimizer (`suggest_optimisation`)

```json
{
    "id": "suggest_optimisation",
      "type": "esql",
      "description": "Recommend better resource allocation to prevent throttling or waste.",
      "tags": [],
      "configuration": {
        "query": """FROM metrics-*
| WHERE @timestamp >= NOW() - 7 days
| WHERE k8s.namespace.name == ?namespace AND (k8s.container.memory_request_utilization IS NOT NULL OR k8s.container.cpu_request_utilization IS NOT NULL)
| STATS 
    avg_cpu_util = AVG(k8s.container.cpu_request_utilization)* 100,
    max_cpu_util = MAX(k8s.container.cpu_request_utilization)* 100,
    p95_cpu_util = PERCENTILE(k8s.container.cpu_request_utilization, 95) * 100,
    avg_mem_percent = AVG(k8s.container.memory_request_utilization)* 100,
    max_mem_percent = MAX(k8s.container.memory_request_utilization)* 100,
    p95_mem_percent = PERCENTILE(k8s.container.memory_request_utilization, 95) * 100
  BY kubernetes.namespace, kubernetes.container.name
| EVAL 
    cpu_recommendation = CASE(
      avg_cpu_util < 30, "Underutilized - Consider reducing CPU requests/limits",
      avg_cpu_util > 80, "Overutilized - Consider increasing CPU requests/limits",
      "Appropriately sized"
    ),
    mem_recommendation = CASE(
      avg_mem_percent < 30, "Underutilized - Consider reducing memory requests/limits",
      avg_mem_percent > 80, "Overutilized - Consider increasing memory requests/limits",
      "Appropriately sized"
    ),
    throttling_risk = CASE(
      max_cpu_util > 90, "High risk of CPU throttling",
      p95_cpu_util > 80, "Medium risk of CPU throttling",
      "Low risk of CPU throttling"
    ),
    oom_risk = CASE(
      max_mem_percent > 90, "High risk of OOM kills",
      p95_mem_percent > 80, "Medium risk of OOM kills",
      "Low risk of OOM kills"
    )
| SORT avg_cpu_util DESC
| LIMIT 100""",
        "params": {
          "namespace": {
            "type": "text",
            "description": "This is the namespace where the pods are failing or for my applications ",
            "optional": false
          }
        }
      },
      "readonly": false
    }
```

**Parameters:**

* `namespace` (Required): The target Kubernetes namespace.

**Logic:**

* **Underutilized:** < 30% avg usage.
* **Overutilized:** > 80% avg usage.
* **OOM Risk:** Max memory usage > 90%.

---

## 📋 Response Format Examples

When the agent executes a tool, it is instructed to return data in a specific Markdown format to ensure the operator can take immediate action.

### Example: Resource Recommendation

> #### 📉 Resource Optimization
> 
> 
> * **Current Configuration:** Request: 512Mi, Limit: 1Gi
> * **Observed Usage:** 950Mi (P95)
> * **Recommendation:** > ```yaml
> resources:
> requests:
> memory: "1Gi"
> limits:
> memory: "1.5Gi"