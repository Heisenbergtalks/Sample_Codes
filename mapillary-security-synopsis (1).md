# Mapillary — Security Synopsis

**To:** Head of Cybersecurity
**Re:** Proposed use of Mapillary street-level imagery for utility pole asset verification
**Basis:** Mapillary Terms of Use, effective 15 February 2024

---

## What we want to do

Use Mapillary's street-level imagery and its pre-detected utility pole locations to find poles missing from our asset records. Mapillary's computer vision already identifies poles and assigns coordinates; we pull those points, compare them against our pole layer, and send the unmatched ones for field verification. A working prototype has confirmed the data exists and the pipeline runs. Proposed next step is a limited pilot in Databricks over a defined test area.

## Vendor and contract

The counterparty is **Meta Platforms Ireland Limited**, not Mapillary AB — Meta acquired Mapillary in 2020. Any existing vendor assessment of Meta may apply. Click-through terms only; no negotiated agreement. Governing law is Sweden, venue is Swedish courts. Data is processed in Sweden, the US and elsewhere. Meta may amend the terms with immediate effect, and continued use constitutes acceptance.

## Access model

Access requires **registering an application on the Mapillary developer dashboard** (mapillary.com/dashboard/developers), which issues a `client_id` and an access token. Each application must have its own `client_id`. The token is a bearer credential passed as a query parameter on every API call — there is no per-user authentication, no scoping, and no expiry we control.

The prototype ran on an individually registered application with a token that was exposed in plaintext during development. **We will destroy that application and register an organizational one before any pilot work**, with the new token held in a Databricks secret scope backed by Key Vault. Rotation ownership will be assigned to the platform team.

Meta reserves the right to throttle usage or revoke the `client_id` at its discretion, and to suspend the account for up to 90 days during any investigation.

## Data flow

Outbound HTTPS only, no inbound exposure. Three destinations: `tiles.mapillary.com` for pole geometry, `graph.mapillary.com` for metadata lookups, and Meta CDN hosts for image thumbnails. The CDN hosts arrive in dynamically signed URLs and are not a stable enumerable list, so strict egress allowlisting will block the imagery stage. If egress cannot be opened for the CDN, we can run geometry-only, which preserves most of the value.

Nothing of ours leaves the environment except the coordinates we query. Note that our query pattern does disclose to Meta which geographies we systematically enumerate, and the terms explicitly authorize Meta to monitor our developer-tool usage and prohibit us from interfering with that monitoring. We assess this as low but non-zero information disclosure, and it is contractually unavoidable.

## Obligations we take on

Three are binding and require work on our side:

**Attribution.** Any view or export that surfaces Mapillary-derived pole data must visibly display the Mapillary logo and link back to Mapillary. This applies to the derived data, not only to images. It becomes a design requirement for the GIS layer and for any exported work order or report.

**Anti-re-identification safeguards.** The commercial-use terms require us to implement and maintain technical safeguards and business processes preventing re-identification or unblurring of any content, including faces and license plates, plus processes preventing inadvertent disclosure. Mapillary blurs faces and plates automatically; our obligation is not to attempt to reverse it and to control the imagery we hold. Any CV work we run must be scoped to poles and documented as excluding people and vehicles.

**Incident notification to Meta.** We must notify `vendor-incident@meta.com` of any incident or activity that could constitute unauthorized or unlawful processing of their content. This is an external notification path to a third party and needs adding to the IR runbook with a defined trigger and owner. It will not happen by default.

## Risk assessment

| Risk | Assessment |
|---|---|
| Service availability | Revocable licence, no SLA, no availability warranty, and the API has been unstable through a backend migration. Acceptable for a discovery tool; not acceptable for any process with an operational or regulatory deadline. |
| Data quality | No fitness-for-purpose warranty. Accuracy is a few metres, poles seen in fewer than three images are silently absent, and imagery in our test area ranges from 2018 to 2025. Supports "this pole may be missing from our records"; does not support any completeness or currency claim. |
| Liability | Capped at $100. We indemnify Meta broadly, including defence costs, with Meta controlling settlement. Standard for free click-through services; means no recourse. |
| Imagery handling | We would store third-party imagery of public rights-of-way that may incidentally contain people and vehicles. Requires classification, access control and a retention period under existing policy. |
| Supply chain | Extraction depends on low-traffic PyPI packages (`mercantile`, `vt2geojson`, `mapbox-vector-tile`). Pin versions, source from internal mirror, run through SCA. |

## Two items requiring Legal determination

These are assigned actions, not open debate:

1. **Commercial-use classification.** Whether our use is "commercial" under the terms determines whether the safeguards and incident-notification obligations above are binding. We are proceeding as though they are, which is the conservative posture. Legal to confirm and record.

2. **Share-alike scope.** Mapillary imagery is CC BY-SA. Whether pole coordinates derived from it carry share-alike into our GIS layer is unsettled in the terms. **Mitigation already designed in:** derived data stays in a separate labelled table with row-level licence and provenance metadata, and no Mapillary-derived value is written to the asset system of record during the pilot. This makes the records identifiable and removable if the determination goes against us, so the pilot can proceed while Legal works.

## Controls for the pilot

- Organizational application registered; prototype credential destroyed
- Token in Databricks secret scope; never in notebook source, output or job logs
- Scope limited to a defined test area, not the full service territory
- Imagery in a Unity Catalog volume with access controls and a set retention period
- Attribution and licence metadata carried as columns on the pole table
- Output treated as candidate leads for field verification only, never as an asset record
- Meta incident notification path defined and owned before imagery is downloaded at scale
- Sequential, rate-limited fetching with backoff; no aggressive parallelism across executors

## Recommendation

Proceed to a limited pilot under the controls above. The security profile is modest: outbound-only, no PII of ours transmitted, no inbound exposure, and the main obligations are implementable. The two Legal items are real but do not block a scoped pilot given the separation and labelling controls.

The failure mode to guard against is scope creep from pilot to production without the safeguards and notification path ever being built. Recommend a checkpoint before any expansion beyond the test area.

**Alternative if this is rejected:** Google Street View is not a substitute — its terms explicitly prohibit deriving utility pole locations from its imagery, prohibit bulk download and storage, and prohibit displaying its imagery alongside non-Google maps, which rules out our Esri-based GIS. The viable paid alternatives are licensed street-level imagery vendors (Cyclomedia, Vexcel, NearMap), state DOT roadway imagery, or our own vehicle capture, which carries no third-party licensing question at all.
