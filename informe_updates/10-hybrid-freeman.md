# Model: hybrid Freeman, freeman-0.4.0 (ADR-0043, finding 2026-10-23-hierarchical)

- **Changed:** the network is scored as Freeman's hierarchy (world → country →
  ASN → IP) with the user's distribution rooted at the population; device / OS /
  browser stay flat; `hour` is out of the score. Replaces 0.3.0 from note 09.
- **New numbers:** ROC-AUC 0.807 → 0.889, recall@1%FPR 0.13 → 0.53; friction
  35.9% → 21.4% at 31/38 stopped; returning users 2–3% friction. Evaluation
  also against Wiefling et al.'s attacker models (naive, VPN, targeted).
- **Review:** model chapter (two features as hierarchies, the paper's
  smoothing, why the published form fails on rare attacker IPs and the rooting
  fix); results chapter (new tables, attacker models); limitations: the dataset
  pseudonymizes values per unique value (Zenodo / Wiefling 2022), so value
  popularity by name does not transfer, which is why the agent stays flat; the
  targeted-attacker-in-same-ISP tail. Cite Wiefling et al. 2022 (ACM TOPS) and
  the dataset DOI 10.5281/zenodo.6782156 if not already.
