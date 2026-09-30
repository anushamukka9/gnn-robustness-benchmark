# gnn-robustness-benchmark

[![CI](https://github.com/anushamukka9/gnn-robustness-benchmark/actions/workflows/ci.yml/badge.svg)](https://github.com/anushamukka9/gnn-robustness-benchmark/actions/workflows/ci.yml)
![Python](https://img.shields.io/badge/python-%3E%3D3.9-blue)
![License](https://img.shields.io/badge/license-MIT-green)

**Adversarial robustness benchmark for Graph Neural Networks.**

A numpy-first evaluation harness implementing the attack suite and metrics
from the companion research on adversarial-robust GNNs
(**TIC 2026, Best Paper**, sole-authored). Compare GNN architectures on
equal footing: same threat model, same budgets, same metrics.

## Companion paper

**"Adversarially Robust Graph Neural Networks for Lateral-Movement
Detection in Enterprise Authentication Logs"**, 3rd IEEE Technology
Innovation Conference (TIC) 2026, Best Paper (Paper ID 734).
Published September 29, 2026: [IEEE Xplore](https://ieeexplore.ieee.org/document/11703008) · DOI [10.1109/TIC68483.2026.11703008](https://doi.org/10.1109/TIC68483.2026.11703008).

> **Published abstract:** We introduce EdgePert-GNN, an adversarially
> robust graph-neural-network framework that detects lateral movement
> by treating enterprise authentication logs as time-windowed,
> multi-relational directed graphs. Authentications are clustered by
> host behavioral roles into communities, and nodes are represented by
> contextual and temporal features engineered to remain stable under
> log perturbations. A relational graph encoder learns role-aware
> embeddings while an edge-perturbation mechanism generates worst-case
> adversarial views at training time, enforcing invariance through a
> contrastive objective. A global graph-pooling head then flags entire
> authentication sessions that indicate attack progression. The
> proposed method advances enterprise threat detection with robustness
> against evasion techniques that commonly defeat signature- and
> sequence-based systems. We evaluated EdgePert-GNN on a benchmark
> authentication dataset containing both benign logins and simulated
> lateral-movement attacks under varying noise and log-manipulation
> conditions. EdgePert-GNN achieves up to 98.6% attack detection
> accuracy at a false-positive rate of 0.8%, and degrades by at most
> 2.1 percentage points under log corruption and adversarial edge
> perturbation.
>
> The published version names the model **EdgePert-GNN** (updated from
> the earlier manuscript's ROBGNN-LMD); the metrics above are the ones
> to cite.

This repo is the runnable companion to that work: the same threat model
and metrics, packaged so anyone can evaluate their own GNN.

## Why

Graph Neural Networks are deployed in fraud detection, recommendation,
and knowledge graphs, all adversarial settings. Yet most GNN papers
report only clean accuracy. This benchmark makes robustness a
first-class, CI-enforceable property:

- **Two attack families**: PGD feature perturbation (L_inf) and
  structural edge-drop (uniform / high-degree)
- **Three metrics**: robust accuracy, attack success rate, and a
  randomized-smoothing certified-radius estimate
- **Zero heavy deps**: numpy + scipy only; torch optional, never required
- **CI gate**: `grbench evaluate` exits non-zero when a model is fragile

## Install

```bash
pip install gnn-robustness-benchmark
```

From source:

```bash
git clone https://github.com/anushamukka9/gnn-robustness-benchmark
cd gnn-robustness-benchmark
pip install -e ".[dev]"
```

## Quickstart

```python
import numpy as np
from gnn_robustness.attacks import pgd_feature_attack, edge_drop_attack
from gnn_robustness.metrics import robust_accuracy, attack_success_rate

x_adv, log = pgd_feature_attack(features, labels, weights, epsilon=0.1, steps=20)
print("robust acc:", robust_accuracy(x_adv, labels, weights))
print("ASR:", attack_success_rate(features, x_adv, labels, weights))

adj_adv, drop_log = edge_drop_attack(adj, drop_fraction=0.1, strategy="high-degree")
print("edges dropped:", drop_log.perturbed_edges)
```

Or run the full suite from the CLI (`.npy`, `.npz`, or CSV inputs):

```bash
grbench evaluate \
  --adj data/adj.npy \
  --features data/features.npy \
  --labels data/labels.npy \
  --weights data/surrogate_weights.npy \
  --epsilon 0.1 --drop-fraction 0.1 --format json
```

See [`examples/quickstart.py`](examples/quickstart.py) for a self-contained
synthetic-graph demo.

## Method

Full protocol details (threat model, PGD hyperparameters, edge-drop
strategies, and the certified-radius estimator) are in
[`docs/METHOD.md`](docs/METHOD.md).

## Dataset: Auth-Vuln-Patch (synthetic v1.0)

> **Synthetic benchmark release. The original AVP records are not published;
> this dataset was generated from the paper's specification (splits, statistics,
> label schema) to preserve the benchmark. Statistics match Table II of the paper.**

**[Download: release avp-v1.0](https://github.com/anushamukka9/gnn-robustness-benchmark/releases/tag/avp-v1.0)**

Auth-Vuln-Patch (AVP) is the companion benchmark from the TIC 2026 Best
Paper: **2,847 labeled authentication incidents** for lateral-movement
detection, adversarially stratified by expert difficulty ratings (1–5).

| Split | Incidents | Unique CVEs | Avg hops | Mean difficulty |
|---|---|---|---|---|
| train | 1,980 | 312 | 7.4 | 2.1 |
| validation | 427 | 89 | 7.1 | 2.2 |
| test-standard | 300 | 74 | 7.8 | 2.0 |
| test-adversarial | 140 | 41 | 9.2 | 4.3 |

Each incident carries an authentication path graph (users/hosts with
features, timestamped auth-event edges), CVE/CWE identifiers, a
ground-truth remediation patch, MITRE ATT&CK sub-technique tags, and one
of five LotL perturbation strategies. Construction sources: LANL-derived
(1,243), MITRE Engenuity incident reports (904), LMDG synthetic (700,
spanning 14 LotL sub-technique variants). License: CC BY 4.0. Full
schema and provenance: [`data/DATASET_CARD.md`](data/DATASET_CARD.md),
[`data/DATA_PROVENANCE.md`](data/DATA_PROVENANCE.md).

```python
import json

with open("auth-vuln-patch-reconstructed-v1.jsonl") as f:
    incidents = [json.loads(line) for line in f]

adv = [i for i in incidents if i["split"] == "test-adversarial"]
print(len(incidents), "incidents |", len(adv), "adversarial test")
print(adv[0]["cve_id"], adv[0]["attck_tags"], adv[0]["perturbation_strategy"])
# -> CVE-2023-14795 ['T1078'] Noisy Decoy  (synthetic identifiers)

# Regenerate the dataset byte-identically from the paper spec:
#   python3 data/generate_avp.py --out-dir data
# Validate against Table II:
#   python3 data/validate_avp.py data/auth-vuln-patch-reconstructed-v1.jsonl
```

## Roadmap

- [ ] White-box attacks against message-passing layers (GCN/GAT gradients)
- [ ] Node-injection attack family
- [ ] Leaderboard harness over Cora / CiteSeer / OGB subsets
- [ ] Export to RobustBench-style model zoo format

Contributions welcome: open an issue with the attack or dataset you need.

## Citation

If you use this benchmark, please cite the companion paper:

Anusha Mukka, "Adversarially Robust Graph Neural Networks for
Lateral-Movement Detection in Enterprise Authentication Logs," 2026
Technology Innovation Conference (TIC), Kuala Lumpur, Malaysia, June
5-6, 2026. DOI: [10.1109/TIC68483.2026.11703008](https://doi.org/10.1109/TIC68483.2026.11703008).

## License

MIT, see [LICENSE](LICENSE).
