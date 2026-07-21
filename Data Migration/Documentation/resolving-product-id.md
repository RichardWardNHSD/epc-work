# Resolving ProductId (Short Code → EPC Product Identifier)

## Overview

The `int_endpoint_templates` and `int_endpoints` DynamoDB tables use short-form `ProductId` values (e.g., `ygm04`, `AC0`, `8hk48`). These are internal codes — the EPC requires an agreed Product ID in the format defined in the [Product ID Mapping](./product-id-mapping.md).

This document defines:
1. The mapping table from short codes to EPC Product Identifiers
2. How to derive the mapping
3. The persistent lookup file used by all migration and delta scripts

---

## Short Code to Product ID Mapping

| Source ProductId (short code) | Supplier Name (from int_endpoints.Name) | Proposed EPC Product ID | Source |
|------|-------------|---------|--------|
| `ygm04` | Cegedim | `CegedimPharmacyServices-v6.0` | product-id-mapping.md #4 |
| `ygm06` | Pharmoutcomes | `PinnaclePharmOutcomes-v2024.12.12` | product-id-mapping.md #9 |
| `ygm17` | Positive Solutions | `HXConsultBaRS-v2.0` | product-id-mapping.md #21 |
| `8hk48` | Sonar | `SonarHealthBaRS-v4.0` | product-id-mapping.md #23 |
| `8JY34` | WASP | `WASPBaRSProvider-v1.0.1` | product-id-mapping.md #28 |
| `AC0` | Advanced | `Adastra-v3.46.00` | product-id-mapping.md #2 |
| `Y01061` | Advanced (alternate instance) | `Adastra-v3.46.00` | product-id-mapping.md #2 (same product, different deployment) |
| `8hq44` | Strata Health | `StrataPathways-v12` | product-id-mapping.md #24 |
| `8ht86` | Fortrus | `AgyleBaRS-v2.6.5` | product-id-mapping.md #11 |
| `RK5` | Nervecentre | `NervecentreBaRS-v9.2` | product-id-mapping.md #17 |
| `RX7` | NWAS | TBD (not in product-id-mapping yet) | Needs adding |
| `GA9` | GMUPCA | TBD (not in product-id-mapping yet) | Needs adding |

---

## How to Derive This Mapping

The `Name` field on `int_endpoints` (and `int_endpoint_templates`) gives the supplier name. Cross-reference this with the Organisation column in the Product ID Mapping table:

```
int_endpoint_templates.ProductId  →  int_endpoint_templates.Name  →  product-id-mapping.Organisation  →  Proposed Product ID
```

For example:
- `ProductId = "ygm04"` has `Name = "Cegedim"` → lookup "Cegedim" in product-id-mapping → `CegedimPharmacyServices-v6.0`
- `ProductId = "8hk48"` has `Name = "Sonar"` → lookup "Sonar" in product-id-mapping → `SonarHealthBaRS-v4.0`

---

## Persistent Lookup File: `product-id-lookup.json`

This mapping must be stored as a persistent file so that it is available to any process (migration, delta detection, validation) regardless of whether they run together or independently.

Store in a shared location (e.g., S3 bucket, repo, or config store) accessible to all scripts:

```json
{
  "ygm04": "CegedimPharmacyServices-v6.0",
  "ygm06": "PinnaclePharmOutcomes-v2024.12.12",
  "ygm17": "HXConsultBaRS-v2.0",
  "8hk48": "SonarHealthBaRS-v4.0",
  "8JY34": "WASPBaRSProvider-v1.0.1",
  "AC0":   "Adastra-v3.46.00",
  "Y01061": "Adastra-v3.46.00",
  "8hq44": "StrataPathways-v12",
  "8ht86": "AgyleBaRS-v2.6.5",
  "RK5":   "NervecentreBaRS-v9.2",
  "RX7":   "TBD-NWAS",
  "GA9":   "TBD-GMUPCA"
}
```

### Loading the lookup

```python
import json

def load_product_id_map(path="product-id-lookup.json"):
    """
    Load the Product ID mapping from a persistent file.
    This file must be maintained by the R&M team and kept in sync
    with the Product ID Mapping document.
    """
    with open(path) as f:
        raw = json.load(f)
    # Normalise keys to uppercase for case-insensitive lookup
    return {k.upper(): v for k, v in raw.items()}

PRODUCT_ID_MAP = load_product_id_map()
```

### Resolving a short code

```python
def resolve_product_id(short_code: str) -> str:
    """Resolve a source ProductId short code to the EPC Product Identifier."""
    product_id = PRODUCT_ID_MAP.get(short_code.upper())
    if not product_id:
        raise ValueError(f"Unknown ProductId: {short_code} — add to product-id-lookup.json")
    if product_id.startswith("TBD"):
        logging.warning(f"ProductId {short_code} mapped to placeholder: {product_id}")
    return product_id
```

---

## Why Persistent?

- The **migration script** (Steps 0–4) uses it to build Templates and Endpoints
- The **delta script** (Step 5) uses it to query the EPC by Product ID and to populate CSV files
- The **validation script** may need it for reporting
- These processes may run at different times, on different machines, or be triggered independently
- A single source of truth avoids drift between processes

---

## Open Questions

| # | Question |
|---|----------|
| 1 | Is `ygm06` (Pharmoutcomes) the same product as `PinnaclePharmOutcomes` or is it a different EMIS product? The endpoint Name says "Pharmoutcomes" but the managing org is EMIS. |
| 2 | `AC0` and `Y01061` both map to Advanced — are these the same Adastra product deployed in different regions, or different products? |
| 3 | `RX7` (NWAS) and `GA9` (GMUPCA) have no entry in the Product ID mapping. These need to be added. |
| 4 | Should short code matching be case-insensitive? (e.g., `8jy34` vs `8JY34`) — current implementation normalises to uppercase. |

---

## Related Documents

| Document | Relationship |
|----------|-------------|
| [Product ID Mapping](./product-id-mapping.md) | Defines the full mapping from supplier/product name to proposed Product ID |
| [Migration: int_ tables → EPC](./migration-process-design.md) | Uses this lookup in Step 2 (Templates) |
| [Migration: targets.json → EPC](./migration-from-targets-json.md) | Uses this lookup in Steps 1, 2, and 5 (delta) |
