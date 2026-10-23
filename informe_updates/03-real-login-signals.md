# Real login signals (ADR-0032)

- **Changed:** client IP comes from trusted proxy headers only; country and ASN
  come from an offline DB-IP Lite database in the image (CC BY 4.0). Demo
  "travel" uses signed simulation tickets, and decisions from them are marked
  `simulado`.
- **Review:** the features/signals section (where IP, country, ASN come from);
  credit DB-IP in references; any text implying signals were synthetic only.
