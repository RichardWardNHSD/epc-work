# Product ID Mapping Table

## Overview

This document maps each BaRS supplier's product name and version (from the "BARS Product
name and version id" spreadsheet) to a proposed Product ID value in the format documented
in [Product ID Format](../Documents/product-id-format.md).

The proposed format follows Option 2 (concatenation) as a fallback until Digital Onboarding
Service-issued Product IDs are confirmed:

```
{ProductName}-v{Version}
```

Where:
- `{ProductName}` is a normalised, hyphenated form of the product name (no spaces, no
  special characters, camelCase or PascalCase)
- `{Version}` is the version string from the supplier

> ⚠️ **These are proposed Product IDs — not confirmed.** The format and values must be
> validated with the Digital Onboarding Service. If formal Product IDs have already been
> issued to these suppliers, those should be used instead. This table is a starting point
> for the NHS team to confirm or replace.

---

## Mapping Table

> **Note:** Only **Receivers** (and Sender+Receiver products) publish Endpoints in the
> Endpoint Catalogue. Sender-only products do not publish Endpoints — they consume the
> catalogue to discover Receiver Endpoints. Sender-only rows are marked with ⚠️ to
> indicate that no Endpoint Template is expected for these products.

| # | Organisation | Product Name | Version | Proposed Product ID | Status | Role | Notes |
|---|-------------|-------------|---------|---------------------|--------|------|-------|
| 1 | ADDVantage Digital Solutions Ltd | healthya | 1 | `Healthya-v1.0` | Development/Assurance | Receiver | |
| 2 | Advanced Health & Care Limited | Adastra | 3.46.00 | `Adastra-v3.46.00` | Live | Sender and Receiver | |
| 3 | Alcidion UK Ltd | Miya Precision (BaRS) | Miya Precision 7.7 | `MiyaPrecisionBaRS-v7.7` | Development/Assurance | Receiver | |
| 4 | Cegedim Rx Ltd. | Pharmacy Services (Cegedim)(BaRS) | V6.0 | `CegedimPharmacyServices-v6.0` | Live | Receiver | |
| 5 | Cleric | Respond-2 | Cleric Interface Service v1.2, Respond-2 v4.7.202 | `ClericRespond2-v4.7.202` | Live | ⚠️ Sender only | No Endpoint expected |
| 6 | Continuum Health Ltd | Anima Health - BaRS | 1.5.11 | `AnimaHealthBaRS-v1.5.11` | Assurance | ⚠️ Sender only | No Endpoint expected |
| 7 | EGTON MEDICAL INFORMATION SYSTEMS LTD | Pinnacle BaRS Service – Local Services | v2024.4 | `PinnacleBaRSLocalServices-v2024.4` | Live | ⚠️ Sender only | No Endpoint expected |
| 8 | EGTON MEDICAL INFORMATION SYSTEMS LTD | Pinnacle BaRS Service - PharmRefer | v.1.3.14 | `PinnacleBaRSPharmRefer-v1.3.14` | Live | ⚠️ Sender only | No Endpoint expected |
| 9 | EGTON MEDICAL INFORMATION SYSTEMS LTD (FULFORD GRANGE) | Pinnacle PharmOutcomes | v2024.12.12 | `PinnaclePharmOutcomes-v2024.12.12` | Live | Receiver | |
| 10 | EMIS Health | EMIS Symphony | BaRS V1.0.0.0 | `EMISSymphony-v1.0.0.0` | Assurance | Receiver | |
| 11 | Fortrus | Agyle (BaRS) | Agyle v2.6.5 | `AgyleBaRS-v2.6.5` | Live | Receiver | |
| 12 | Harris | Flex | Harris Flex version 6.4.2 | `HarrisFlex-v6.4.2` | Development/Assurance | Receiver | |
| 13 | Humber Teaching NHS Foundation Trust | Interweave-BaRS | 2.8.0 | `InterweaveBaRS-v2.8.0` | Supplier assured | Receiver | |
| 14 | IC24 | CLEO Core | CLEO Core 2.0 | `CLEOCore-v2.0` | Development/Assurance | Sender and Receiver | |
| 15 | Medicus Health Ltd | Medicus - BaRS | 1.6 | `MedicusBaRS-v1.6` | Development | ⚠️ Sender only | No Endpoint expected |
| 16 | MIS Emergency Systems Ltd | Tripath BaRS | 23.1.0.12 | `TripathBaRS-v23.1.0.12` | Live | ⚠️ Sender only | No Endpoint expected |
| 17 | Nervecentre Software Ltd | Nervecentre BaRS | 9.2 | `NervecentreBaRS-v9.2` | Supplier assured | Receiver | |
| 18 | NHS Digital: NHS 111 Online | NHS 111 Online - BaRS | v1.4.0 | `NHS111OnlineBaRS-v1.4.0` | Live | ⚠️ Sender only | No Endpoint expected |
| 19 | NHSE - EDS | Streaming & Redirection - BaRS API | 1.0.0 | `StreamingRedirectionBaRS-v1.0.0` | Live | ⚠️ Sender only | No Endpoint expected |
| 20 | OX. Digital Health | OX.DH (BaRS, DoS UEC - REST API) | 1.9.4 | `OXDHBaRS-v1.9.4` | Development | ⚠️ Sender only | No Endpoint expected |
| 21 | Positive Solutions | HXConsult - BaRS | 2.0 | `HXConsultBaRS-v2.0` | Live | Receiver | |
| 22 | Quicksilva Ltd | conneQt Platform - BaRS | 1.0 | `ConneQtPlatformBaRS-v1.0` | Assurance | Receiver | |
| 23 | Sonar Informatics Limited | SonarHealth - BaRS | v4.0 | `SonarHealthBaRS-v4.0` | Live | Receiver | |
| 24 | STRATA HEALTH UK LTD | Pathways | Strata Pathways v12 | `StrataPathways-v12` | Live | Receiver | |
| 25 | Swiftqueue | Swiftqueue(Dedalus) NHS BaRS | 4.2 | `SwiftqueueBaRS-v4.2` | Assurance | Receiver | |
| 26 | Invatech Health | Titanverse - BaRS | Titanverse 3.1 | `TitanverseBaRS-v3.1` | Development/Assurance | Receiver | |
| 27 | The Phoenix Partnership (TPP) | SystmOne | MR224 | `SystmOne-vMR224` | Live | ⚠️ Sender only | No Endpoint expected |
| 28 | WASP | WASP BaRS Provider | WASP BaRS Provider Service 1.0.1 | `WASPBaRSProvider-v1.0.1` | Live | Receiver | |

---

## Notes on the proposed IDs

### Format rules applied

1. Spaces removed, words concatenated in PascalCase
2. Special characters removed (parentheses, ampersands, dots in names)
3. "BaRS" retained where it's part of the product identity (distinguishes from non-BaRS products)
4. Version prefix normalised: `v` prefix added if not present, leading/trailing spaces trimmed
5. Where multiple version numbers exist in the source (e.g., row 5), the primary product version is used

### Open questions for the NHS team

| # | Question |
|---|----------|
| 1 | Should the version be included in the Product ID? The [Product ID Format](../Documents/product-id-format.md) recommends version-agnostic IDs (e.g., `PinnaclePharmOutcomes` without `-v2024.12.12`). If version-agnostic is adopted, the `-v{Version}` suffix should be removed from all proposed IDs. |
| 2 | Have any of these suppliers already been issued a Product ID by the Digital Onboarding Service? If so, those formal IDs should replace the proposed values in this table. |
| 3 | Row 5 (Cleric) has two version numbers — which is the authoritative product version? |
| 4 | Row 7 and Row 8 (EGTON/Pinnacle) — are these the same product with different capabilities, or genuinely separate products? If the same product, they should share a Product ID with different `connectionType`/`payloadType` on their Templates. |
| 5 | Row 9 (Pinnacle PharmOutcomes) — is this a separate product from rows 7/8, or the same Pinnacle product deployed as a Receiver? |
| 6 | Should suppliers in "Development/Assurance" status receive Product IDs now, or only once they reach "Live"? |

---

## Version-agnostic alternative

If the team adopts the **version-agnostic** recommendation from the Product ID Format
document, the Product IDs would be:

| # | Organisation | Proposed Product ID (version-agnostic) | Role |
|---|-------------|---------------------------------------|------|
| 1 | ADDVantage Digital Solutions Ltd | `Healthya` | Receiver |
| 2 | Advanced Health & Care Limited | `Adastra` | Sender and Receiver |
| 3 | Alcidion UK Ltd | `MiyaPrecisionBaRS` | Receiver |
| 4 | Cegedim Rx Ltd. | `CegedimPharmacyServices` | Receiver |
| 5 | Cleric | `ClericRespond2` | ⚠️ Sender only |
| 6 | Continuum Health Ltd | `AnimaHealthBaRS` | ⚠️ Sender only |
| 7 | EGTON MEDICAL INFORMATION SYSTEMS LTD | `PinnacleBaRSLocalServices` | ⚠️ Sender only |
| 8 | EGTON MEDICAL INFORMATION SYSTEMS LTD | `PinnacleBaRSPharmRefer` | ⚠️ Sender only |
| 9 | EGTON MEDICAL INFORMATION SYSTEMS LTD (FULFORD GRANGE) | `PinnaclePharmOutcomes` | Receiver |
| 10 | EMIS Health | `EMISSymphony` | Receiver |
| 11 | Fortrus | `AgyleBaRS` | Receiver |
| 12 | Harris | `HarrisFlex` | Receiver |
| 13 | Humber Teaching NHS Foundation Trust | `InterweaveBaRS` | Receiver |
| 14 | IC24 | `CLEOCore` | Sender and Receiver |
| 15 | Medicus Health Ltd | `MedicusBaRS` | ⚠️ Sender only |
| 16 | MIS Emergency Systems Ltd | `TripathBaRS` | ⚠️ Sender only |
| 17 | Nervecentre Software Ltd | `NervecentreBaRS` | Receiver |
| 18 | NHS Digital: NHS 111 Online | `NHS111OnlineBaRS` | ⚠️ Sender only |
| 19 | NHSE - EDS | `StreamingRedirectionBaRS` | ⚠️ Sender only |
| 20 | OX. Digital Health | `OXDHBaRS` | ⚠️ Sender only |
| 21 | Positive Solutions | `HXConsultBaRS` | Receiver |
| 22 | Quicksilva Ltd | `ConneQtPlatformBaRS` | Receiver |
| 23 | Sonar Informatics Limited | `SonarHealthBaRS` | Receiver |
| 24 | STRATA HEALTH UK LTD | `StrataPathways` | Receiver |
| 25 | Swiftqueue | `SwiftqueueBaRS` | Receiver |
| 26 | Invatech Health | `TitanverseBaRS` | Receiver |
| 27 | The Phoenix Partnership (TPP) | `SystmOne` | ⚠️ Sender only |
| 28 | WASP | `WASPBaRSProvider` | Receiver |

---

## Next steps

1. **NHS team to review** this mapping and confirm or correct each proposed Product ID
2. **Contact the Digital Onboarding Service** to check if formal Product IDs already exist
   for these suppliers
3. **Decide version-inclusive vs version-agnostic** — this determines which column above
   becomes the definitive mapping
4. **Publish the confirmed mapping** — this table becomes the input to the data migration
   pipeline (Step 1: Template creation)
