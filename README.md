<!--
  MAINTAINING THIS README
  - This page is an overview: banner, architecture, one card per phase, cleanup.
  - Step-by-step commands live in each phase folder's README.md (manifests/<phase>/README.md).
  - Screenshots live in screenshots/; encode spaces in file names as %20.
  - Header, footer, typing line, badges and icons come from capsule-render.vercel.app,
    readme-typing-svg.demolab.com, shields.io and skillicons.dev.
-->

<a id="top"></a>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:0F172A,50:7B42BC,100:FF9900&height=300&section=header&text=Generative%20AI%20on%20Amazon%20EKS&fontSize=46&fontColor=FFFFFF&fontAlignY=38&animation=fadeIn&desc=From%20an%20empty%20AWS%20account%20to%20agents%2C%20cache%20offloading%20and%20RAG%20%E2%80%94%20on%20one%20GPU&descSize=18&descAlignY=58" width="100%" alt="Generative AI on Amazon EKS"/>

<div align="center">

<a href="#the-phases"><img src="https://readme-typing-svg.demolab.com?font=Fira+Code&weight=600&size=24&duration=2800&pause=1000&color=FF9900&center=true&vCenter=true&width=760&height=50&lines=Serve+Ministral+8B+with+vLLM+on+one+A10G;Watch+the+GPU+and+the+engine+in+Grafana;Give+the+model+tools+with+Strands;Cut+time+to+first+token+16x+with+LMCache;Ground+answers+in+your+own+data+with+RAG" alt="What this repo does"/></a>

<br/><br/>

<img src="https://img.shields.io/badge/Amazon_EKS-Auto_Mode-FF9900?style=for-the-badge&logo=amazoneks&logoColor=white&labelColor=0F172A" alt="Amazon EKS"/>
<img src="https://img.shields.io/badge/Terraform-IaC-7B42BC?style=for-the-badge&logo=terraform&logoColor=white&labelColor=0F172A" alt="Terraform"/>
<img src="https://img.shields.io/badge/vLLM-0.21-30A2FF?style=for-the-badge&labelColor=0F172A" alt="vLLM"/>
<img src="https://img.shields.io/badge/NVIDIA-A10G_24GB-76B900?style=for-the-badge&logo=nvidia&logoColor=white&labelColor=0F172A" alt="NVIDIA A10G"/>
<img src="https://img.shields.io/badge/Grafana-Observability-F46800?style=for-the-badge&logo=grafana&logoColor=white&labelColor=0F172A" alt="Grafana"/>
<img src="https://img.shields.io/badge/Strands-Agents-B3B3B3?style=for-the-badge&logo=python&logoColor=white&labelColor=0F172A" alt="Strands"/>

<br/><br/>

<table>
<tr>
<td align="center" width="170"><h2>7</h2><sub>phases, from Terraform<br/>to RAG</sub></td>
<td align="center" width="170"><h2>250 ms</h2><sub>time to first token<br/>at baseline</sub></td>
<td align="center" width="170"><h2>2.2 req/s</h2><sub>sustained on<br/>one A10G</sub></td>
<td align="center" width="170"><h2>16.4×</h2><sub>faster first token<br/>with LMCache</sub></td>
</tr>
</table>

<sub><code>Ministral-3-8B-Instruct-2512 (BF16)</code> &nbsp;·&nbsp; <code>ap-south-1</code> &nbsp;·&nbsp; <code>genai-workshop</code> &nbsp;·&nbsp; <code>g5.2xlarge</code></sub>

</div>

<br/>

<p align="center"><img src="./architecture.png" alt="Architecture" width="85%"/></p>

## The phases

### 01 · Cluster Setup

<sub><code>terraform/</code></sub>

Terraform builds everything the later phases run on: an EKS Auto Mode cluster, the VPC, an S3 bucket holding the model weights, Amazon Managed Prometheus and Grafana. We ran it in `ap-south-1` on **g5.2xlarge (NVIDIA A10G, 24 GB)** nodes. That GPU can't run the FP8 checkpoint correctly, so we serve the **BF16** model instead.

**[Step-by-step guide →](./terraform/README.md)**

<br/>

### 02 · Serve a Model

<sub><code>manifests/100-vllm</code></sub>

vLLM streams Ministral 8B from S3 onto the GPU and serves it through an OpenAI-compatible API. Open WebUI puts a chat interface in front of it. The NVIDIA DCGM Exporter and a vLLM ServiceMonitor feed Prometheus, so Grafana shows both GPU health and inference metrics: token rates, time to first token and time per output token.

<table>
<tr>
<td align="center" valign="top"><a href="./screenshots/vllm-test.png"><img src="./screenshots/vllm-test.png" width="220" alt="First tokens"/></a><br/><sub>First tokens over the API</sub></td>
<td align="center" valign="top"><a href="./screenshots/open-webui.png"><img src="./screenshots/open-webui.png" width="220" alt="Open WebUI"/></a><br/><sub>Chatting in Open WebUI</sub></td>
<td align="center" valign="top"><a href="./screenshots/nvidia-dgcm.png"><img src="./screenshots/nvidia-dgcm.png" width="220" alt="DCGM dashboard"/></a><br/><sub>GPU at 52 °C / 214 W peak</sub></td>
<td align="center" valign="top"><a href="./screenshots/vllm-metrics.png"><img src="./screenshots/vllm-metrics.png" width="220" alt="vLLM dashboard"/></a><br/><sub>vLLM engine metrics</sub></td>
</tr>
</table>

**[Step-by-step guide →](./manifests/100-vllm/README.md)**

<br/>

### 03 · Agentic App

<sub><code>manifests/200-strands-agent</code></sub>

A Strands agent, packaged as a FastAPI service, uses the vLLM model as its reasoning engine and has two tools: one for current time and one for live weather. Each question takes two model calls. In the first, the model chooses a tool; in the second, it turns the tool's result into a natural-language answer.

<table>
<tr>
<td align="center" valign="top"><a href="./screenshots/agents-in-action.png"><img src="./screenshots/agents-in-action.png" width="460" alt="Agent in action"/></a><br/><sub>Live time in Dallas, weather in Denver</sub></td>
</tr>
</table>

**[Step-by-step guide →](./manifests/200-strands-agent/README.md)**

<br/>

### 04 · Benchmarking

<sub><code>manifests/300-benchmarking</code></sub>

We load-tested the model with a Helm-packaged benchmark job and watched it on a dedicated Grafana dashboard. The **baseline** run (1 req/s, 512-token prompts) completed every request with a 250 ms mean time to first token and 45 ms per output token. The **saturation** run stepped from 5 to 20 req/s. The A10G topped out at about 2.2 req/s (roughly 550 output tokens/s). Beyond that, the KV cache stayed near 100%, requests piled up in the queue, and 22.5% timed out at the highest rate.

<table>
<tr>
<td align="center" valign="top"><a href="./screenshots/baseline-scenerio.png"><img src="./screenshots/baseline-scenerio.png" width="220" alt="Baseline benchmark"/></a><br/><sub>Baseline: 60 / 60 succeeded</sub></td>
<td align="center" valign="top"><a href="./screenshots/saturation-scenerio.png"><img src="./screenshots/saturation-scenerio.png" width="220" alt="Saturation benchmark"/></a><br/><sub>Saturation: 5 → 20 req/s</sub></td>
<td align="center" valign="top"><a href="./screenshots/Screenshot%202026-10-12%20113503.png"><img src="./screenshots/Screenshot%202026-10-12%20113503.png" width="220" alt="Latency panels"/></a><br/><sub>Throughput and latency</sub></td>
</tr>
<tr>
<td align="center" valign="top"><a href="./screenshots/benchmark-engine-metrics-1.png"><img src="./screenshots/benchmark-engine-metrics-1.png" width="220" alt="Engine metrics"/></a><br/><sub>Cache and queue under load</sub></td>
<td align="center" valign="top"><a href="./screenshots/becnhmarking-engine-metrics-2.png"><img src="./screenshots/becnhmarking-engine-metrics-2.png" width="220" alt="Scheduler state"/></a><br/><sub>Scheduler state and request lengths</sub></td>
<td align="center" valign="top"><a href="./screenshots/benchmarking-vllm-metrics.png"><img src="./screenshots/benchmarking-vllm-metrics.png" width="220" alt="vLLM metrics under load"/></a><br/><sub>Preemptions climbing</sub></td>
</tr>
</table>

**[Step-by-step guide →](./manifests/300-benchmarking/README.md)**

<br/>

### 05 · KV Cache Offloading

<sub><code>manifests/400-lmcache</code></sub>

LMCache keeps a prompt's KV tensors after the first request, so a repeat request with the same prefix skips most of the prefill work. With **L1** (CPU RAM on the same pod), time to first token for a 4K-token prompt dropped from 4.46 s to 0.27 s, **16.4× faster**. Adding **L2** (Valkey, shared across pods) took it from 5.78 s to 0.50 s, **11.6× faster**, and new pods can reuse the cache too.

<table>
<tr>
<td align="center" valign="top"><a href="./screenshots/LMCache-L1.png"><img src="./screenshots/LMCache-L1.png" width="340" alt="LMCache L1"/></a><br/><sub>L1 CPU RAM: 16.4× faster TTFT</sub></td>
<td align="center" valign="top"><a href="./screenshots/LMCache%20L1%20%2B%20L2-test.png"><img src="./screenshots/LMCache%20L1%20%2B%20L2-test.png" width="340" alt="LMCache L1 + L2"/></a><br/><sub>L1 + L2 Valkey: 11.6× faster TTFT</sub></td>
</tr>
</table>

**[Step-by-step guide →](./manifests/400-lmcache/README.md)**

<br/>

### 06 · Fine-Tuning

<sub><code>manifests/600-finetuning</code></sub>

LoRA fine-tuning on a startup-advisor dataset, then serving the adapter next to the base model in vLLM. We changed the training job to load the BF16 checkpoint directly, so it fits a single 24 GB A10G.

**[Step-by-step guide →](./manifests/600-finetuning/README.md)**

<br/>

### 07 · RAG

<sub><code>manifests/700-rag</code></sub>

A retrieval-augmented generation pipeline: a job embeds a product catalogue into an S3 vector store, and a Gradio app answers the same question twice, once from the model alone and once with retrieved context. Side by side, the RAG answer cites the actual products and their similarity scores, while the plain model answers in generalities.

<table>
<tr>
<td align="center" valign="top"><a href="./screenshots/RAG-test.png"><img src="./screenshots/RAG-test.png" width="340" alt="RAG comparison"/></a><br/><sub>Laptops: generic vs grounded</sub></td>
<td align="center" valign="top"><a href="./screenshots/RAG-test-2.png"><img src="./screenshots/RAG-test-2.png" width="340" alt="RAG with retrieved context"/></a><br/><sub>Smart bulbs, with retrieved context</sub></td>
</tr>
</table>

**[Step-by-step guide →](./manifests/700-rag/README.md)**

<br/>

## Cleanup

> [!CAUTION]
> **Leave no GPU running.** g5 nodes are billed by the hour, whether or not you're using them.

<table>
<tr>
<td width="50%" valign="top">

**1 · Tear it all down**

```bash
cd terraform
terraform destroy --auto-approve
```

</td>
<td width="50%" valign="top">

**2 · Confirm it's gone**

```bash
aws eks list-clusters --region ap-south-1
# genai-workshop should no longer be listed
```

</td>
</tr>
</table>

<br/>

<div align="center">

### The journey, in one line

[Cluster Setup](./terraform/README.md) &nbsp;→&nbsp;
[Serve a Model](./manifests/100-vllm/README.md) &nbsp;→&nbsp;
[Agentic App](./manifests/200-strands-agent/README.md) &nbsp;→&nbsp;
[Benchmarking](./manifests/300-benchmarking/README.md) &nbsp;→&nbsp;
[KV Cache Offloading](./manifests/400-lmcache/README.md) &nbsp;→&nbsp;
[Fine-Tuning](./manifests/600-finetuning/README.md) &nbsp;→&nbsp;
[RAG](./manifests/700-rag/README.md)

<br/>

<sub>BUILT WITH</sub>

<img src="https://skillicons.dev/icons?i=aws,kubernetes,terraform,docker,python,prometheus,grafana&perline=7" alt="AWS, Kubernetes, Terraform, Docker, Python, Prometheus, Grafana"/>

<br/>

<sub>plus <a href="https://docs.vllm.ai">vLLM</a> · <a href="https://strandsagents.com">Strands Agents</a> · <a href="https://docs.lmcache.ai">LMCache</a> · <a href="https://docs.openwebui.com">Open WebUI</a> · <a href="https://github.com/NVIDIA/dcgm-exporter">NVIDIA DCGM</a></sub>

</div>

<br/>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:FF9900,50:7B42BC,100:0F172A&height=170&section=footer&text=Seven%20phases.%20One%20GPU.&fontSize=30&fontColor=FFFFFF&fontAlignY=70&animation=fadeIn" width="100%" alt="Seven phases. One GPU."/>

<p align="center">
<a href="#top"><b>Back to top ↑</b></a> &nbsp;·&nbsp;
<a href="https://catalog.workshops.aws/genai-on-eks/en-US">Workshop</a> &nbsp;·&nbsp;
<a href="CONTRIBUTING.md#security-issue-notifications">Security</a> &nbsp;·&nbsp;
<a href="LICENSE">MIT-0 License</a>
</p>
