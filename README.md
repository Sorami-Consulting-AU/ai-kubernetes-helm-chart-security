# AI on Kubernetes: Helm chart security defaults (evidence)

This repository holds the data and evidence behind the Sorami technical report **"AI on Kubernetes: 10 of 14 Helm charts ship with no API auth"** (version 1.2).

**Read the report here:** https://sorami.com.au/research/ai-kubernetes-helm-chart-security/

The method, findings, severity ratings and hardening advice are in the report. This repository does not repeat them. It holds the data files the report cites, so readers can check each claim.

## Charts tested

Default values, rendered 24 September 2026. Selected live probes were rerun on 25 September 2026.

| Chart | Version | Source |
|---|---|---|
| vllm/vllm-stack | 0.1.12 | https://vllm-project.github.io/production-stack |
| kserve-resources | v0.20.0 | oci://ghcr.io/kserve/charts/kserve-resources |
| kuberay/kuberay-operator, kuberay/ray-cluster | 1.7.1 | https://ray-project.github.io/kuberay-helm/ |
| vllm-project/aibrix dist/chart | 0.7.0 | git tag v0.7.0 |
| otwld/ollama | 1.83.0 | https://helm.otwld.com/ |
| open-webui/open-webui | 16.6.0 | https://helm.openwebui.com/ |
| go-skynet/local-ai | 3.4.2 | https://go-skynet.github.io/helm-charts/ |
| litellm-helm | 1.102.1 | oci://ghcr.io/berriai/litellm-helm |
| langfuse/langfuse | 2.1.2 | https://langfuse.github.io/langfuse-k8s |
| n8n-io/n8n-hosting charts/n8n | 1.13.0 | git commit 359e1772 |
| qdrant/qdrant | 1.19.1 | https://qdrant.github.io/qdrant-helm |
| weaviate/weaviate | 17.8.3 | https://weaviate.github.io/weaviate-helm |
| milvus/milvus | 5.0.28 | https://zilliztech.github.io/milvus-helm/ |
| containers/kubernetes-mcp-server | 0.1.0 | oci://ghcr.io/containers/charts/kubernetes-mcp-server |
| Flux159/mcp-server-kubernetes helm-chart | 2.8.0 | git tag v4.1.7 |

## Files

- `results.csv`: one row per chart, one column per check, with the evidence for each value.
- `tools.tsv`: chart sources, pinned versions and the render override (if any) each chart needed.
- `manual-checks.tsv`: API auth and telemetry defaults, checked by hand against vendor docs and source.
- `dynamic-results.tsv`: results of the live probes on a local test cluster.
- `evidence/totals.json`: the headline counts used in the report.
- `evidence/doc-urls.tsv`: vendor documentation cited for each claim, with HTTP status on 24 September 2026.
- `evidence/scans/scan-summary.json`, `evidence/scans/scan-detail.json`: per-chart counts from Kubescape 4.0.14, Checkov 3.3.19, kube-linter 0.8.3 and Trivy 0.74.0.
- `evidence/dynamic/`: request and response logs from the live probes, and `kubectl auth can-i` output for the ServiceAccounts discussed in the report.
- `figures/`: the figures used in the report.

## How to read the data

- A `file:line` reference such as `values.yaml:17` points into the chart's own files at the pinned version above. Paths like `charts/repo-qdrant/qdrant/values.yaml` are relative to the unpacked chart.
- `off`, `on`, `not_enforced` and similar values are defined in the report.
- In the probe logs, the kubeconfig path, kube context, chart cache path and cluster IP addresses are replaced with placeholders such as `<kubeconfig>`, `<kind-context>`, `<charts>` and `<pod-ip>`. Nothing else in the request or response was changed.
- A random password that the PostgreSQL subchart generated at install time is replaced with `REDACTED-RENDER-TIME-PASSWORD`. Placeholder credentials that vendors ship in their charts are left in place, because they are the finding.

## Reproduce the rendered manifests

We do not redistribute vendor chart files or rendered manifests. Anyone can regenerate them from the public charts:

```bash
helm repo add vllm https://vllm-project.github.io/production-stack
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm repo add otwld https://helm.otwld.com/
helm repo add open-webui https://helm.openwebui.com/
helm repo add langfuse https://langfuse.github.io/langfuse-k8s
helm repo add qdrant https://qdrant.github.io/qdrant-helm
helm repo add weaviate https://weaviate.github.io/weaviate-helm
helm repo add milvus https://zilliztech.github.io/milvus-helm/
helm repo add localai https://go-skynet.github.io/helm-charts/
helm repo update

helm template rel vllm/vllm-stack --version 0.1.12 --namespace sbd
helm template rel kuberay/kuberay-operator --version 1.7.1 --namespace sbd
helm template rel kuberay/ray-cluster --version 1.7.1 --namespace sbd
helm template rel otwld/ollama --version 1.83.0 --namespace sbd
helm template rel open-webui/open-webui --version 16.6.0 --namespace sbd
helm template rel langfuse/langfuse --version 2.1.2 --namespace sbd --set clickhouse.crdCheck=false
helm template rel qdrant/qdrant --version 1.19.1 --namespace sbd
helm template rel weaviate/weaviate --version 17.8.3 --namespace sbd
helm template rel milvus/milvus --version 5.0.28 --namespace sbd
helm template rel localai/local-ai --version 3.4.2 --namespace sbd
helm template rel oci://ghcr.io/berriai/litellm-helm --version 1.102.1 --namespace sbd
helm template rel oci://ghcr.io/kserve/charts/kserve-resources --version v0.20.0 --namespace sbd
helm template rel oci://ghcr.io/containers/charts/kubernetes-mcp-server --version 0.1.0 --namespace sbd --set ingress.host=mcp.example.invalid

git clone --depth 1 --branch v0.7.0 https://github.com/vllm-project/aibrix.git
helm template rel aibrix/dist/chart --namespace sbd

git clone --depth 1 --branch v4.1.7 https://github.com/Flux159/mcp-server-kubernetes.git
helm template rel mcp-server-kubernetes/helm-chart --namespace sbd

git clone https://github.com/n8n-io/n8n-hosting.git && git -C n8n-hosting checkout 359e1772f9e4987c964aa4d6ab1621d07bdb107f
helm template rel n8n-hosting/charts/n8n --namespace sbd --set secretRefs.env.N8N_ENCRYPTION_KEY=research-placeholder-not-a-secret
```

Use Helm 3.19 or later (the Langfuse chart needs `fromToml`). No kube context is needed to render. Some charts use floating image tags, so a later render may reference newer images.

## Disclosure

Every finding is a default that the vendor documents, and the report links to the vendor's own documentation for each one. Testing ran only on a local cluster created for the study and deleted afterwards. No third-party, customer or internet-facing system was scanned or contacted. Vendors with a question about a finding can write to hello@sorami.com.au.

## Licence

The data, evidence files and figures in this repository are released under CC BY 4.0. Copyright 2026 Sorami Consulting Pty Ltd. See `LICENSE`.

Cite as: Sorami (2026). *AI on Kubernetes: 10 of 14 Helm charts ship with no API auth.* Sorami Technical Report, version 1.2. https://sorami.com.au/research/ai-kubernetes-helm-chart-security/

Contact: hello@sorami.com.au
