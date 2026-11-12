# SUDOKU on Stock – Fiori apps (Tradin)

Implementation of *Functional Design SUDOKU V0.2* (requirement 3-2697): three SAP Fiori apps on an
SAP S/4HANA 2025 on‑premise **OData V4** backend, plus a CAP mock that materialises the service contract.

| Path | What it is |
|---|---|
| `apps/sudoku-blending` | **Order Cockpit** (FE List Report) → **Blending Workbench** (FPM custom page) with **Proposal Comparison** dialog, release / force release / reopen, audit log |
| `apps/sudoku-specification-profiles` | Specification Profiles maintenance (FE List Report + Object Page, draft, child characteristics) – FR‑14 |
| `apps/sudoku-operational-rules` | Operational Rules maintenance (FE List Report + Object Page, draft) – FR‑10/14 |
| `contract/metadata.xml` | The OData V4 service contract (`$metadata`, including UI annotations) the apps are built against |
| `backend-mock/` | CAP project that implements the contract with real logic (rule resolution, blend calculation, hard validation, proposals, release). **Dev only – nothing here ships.** |
| `docs/service-contract.md` | Entities, actions, annotations and calculation rules – the specification for the ABAP/RAP team |

## Run it locally (mock backend)

```bash
npm install                      # once, installs all workspaces
npm run mock:backend             # CAP mock on :4004, served at the same path as the on-prem service
npm run start:blending           # in a second terminal – opens the FLP sandbox
npm run start:profiles
npm run start:rules
```

The mock serves `/sap/opu/odata4/sap/zui_sudoku_o4/srvd/sap/zui_sudoku/0001/`, i.e. exactly the path the apps use against
the real system, so switching between mock and on‑prem only changes the proxy target. Data is in‑memory and reset on restart.

Demo orders follow the figures of the functional design:

| Order | Material group | Status | What to look at |
|---|---|---|---|
| 4500019255 | Cocoa (Fairtrade) | Proposal in Progress | Cd 0.64 mg/kg > hard limit 0.60 → red banner, not releasable; thin pool → proposals flagged "insufficient stock" (Fig. 2/4) |
| 4500019240 | Cocoa (Organic, premium material) | Proposed | `COC-MASS-PREMIUM-OVERRIDE` wins over the plant override (fallback hierarchy) |
| 4500019260 | Cocoa (RA/UTZ) | Open | Start → generate proposals → adopt → release |
| 4500019012 | Orange Juice | Released | Brix 11.92 / acidity 0.75 / ratio 15.87, compliant (Fig. 3); components written back |
| 4500019310 / 4500019315 | Orange Juice | Open / In progress | Same workbench, Juice characteristics resolved from the profile (FR‑15) |
| 4500019301 | Coffee | Open | No profile configured → actions disabled |

## Run against the on‑premise system

1. In each app's `ui5.yaml` replace `https://s4h.tradin.example.com` (placeholder) with the ABAP system host
   (and add `client:` under `backend` if needed). Credentials are taken from the Fiori tools system store
   (`npx fiori add ...` / SAP Fiori tools "Save credentials"), never from the repo.
2. Publish the OData V4 service (service binding for the definition in `docs/service-contract.md`) under the path
   `/sap/opu/odata4/sap/zui_sudoku_o4/srvd/sap/zui_sudoku/0001/`. If the real path differs, change `dataSources.mainService.uri`
   in each `manifest.json`.
3. `npm run start -w sudoku-blending` (live backend). Deployment to the ABAP repository: `npm run deploy-config -w <app>`.
4. Refresh the local copy of the metadata once the real service exists (see the `sap-fiori-create-cli` skill,
   *update service-metadata*) – `webapp/localService/mainService/metadata.xml` is read‑only tooling input.

> **Important:** the UI annotations (list columns, filters, object‑page facets, value helps, feature control of the actions) are
> part of the contract and are expected from the backend (CDS metadata extensions). They are listed in `docs/service-contract.md`
> and implemented in `backend-mock/srv/annotations.cds`. If the backend does not deliver them, add them to each app's
> `webapp/annotations/annotation.xml`.

## Requirement traceability

| FR | Implementation |
|---|---|
| FR‑01/02/04 | Cockpit lists ZSU orders; header = one blend; statuses Open, Proposal in Progress, Proposed, Released, Executed, Cancelled with semantic colours; filters plant, subcontractor, material, material group, delivery date, status, search |
| FR‑03/11 | `release` creates one final batch, writes it to **all** items and creates the subcontracting components (`Components`), visible in *Write‑back* section |
| FR‑05 | Batch pool table with editable quantity per batch (+ "use maximum" / clear); every change is PATCHed and recalculated by the backend |
| FR‑06/07 | *Generate Proposal* (priorities dialog) → three ranked proposals from eligible stock only (status released, stock > 0, shelf life) |
| FR‑08/09/15 | `Orders.ProfileId` resolved by the fallback hierarchy customer+subcontractor+material → … → default, **whole profile wins**; the workbench renders whatever characteristics/groups/limits the profile defines – no material‑group logic in the UI |
| FR‑10 | Operational rules: max source batches, min quantity, min contribution %, rounding – enforced in validation and in proposal generation |
| FR‑12 | Force release dialog: lists the overridable issues, requires the explicit override, reason field; audit log stores user, timestamp, force flag, reason, calculated snapshot |
| FR‑14 | Both configuration apps: draft handling, inline creation of characteristics, value helps, validations (min ≤ target ≤ max, dates, …) |
| §5.2.7 | Actions are enabled/disabled by `Core.OperationAvailable` flags computed along the status model; reopen of a released blend withdraws final batch and components |
| §5.2.5.4/5.3 | Hard validation (spec, safety margin, stock, min qty, rounding, max batches, min contribution, unusable batch) as message strips; *forceable* vs *non‑forceable* distinguished |
| FR‑13 | Not in MVP scope (as in the FD) |

Out of scope per FD §4.1/§9: Cocoa Batch Master app, Characteristic Profile, Optimization Profile, Status Configuration
(the characteristic catalog and the default proposal presets exist in the mock, without a maintenance app).

## Decisions worth knowing

* **The metadata that came with the request was not the SUDOKU service.** It is the generic SAP Gateway *common* service
  (`Currencies`, `UnitsOfMeasure`, `DocumentDescriptions`, `T100Longtexts`) that every V4 service group exposes under
  `…/default/iwbep/common/0001/`. The SUDOKU service contract was therefore derived from the functional design (`contract/metadata.xml`).
  Replace it with the real `$metadata` as soon as it is available and align differences (entity/property names) in the views.
* Units (`QuantityUnit`, currency) use local value‑list entities in the mock. On the real system point them to the common
  service (`UnitsOfMeasure`, `Currencies`) with `Common.ValueListReferences`.
* Authorizations (§4.3 roles Viewer / Blender / Releaser / Configuration Owner) are enforced by the backend; the UI only
  reacts to the operation‑availability flags. The mock has no role checks.
* Quantities in the workbench are edited per batch; validation is authoritative in the backend (the UI never calculates).

## Status of verification

Verified in a browser against the mock: cockpit, workbench for cocoa and orange juice orders, recalculation after quantity
change, hard‑validation banner, proposal generation / comparison / adoption, save as proposed, release, forced release (dialog and
audit entry), reopen, profile list / object page / draft edit + save (activation), operational rules list.
Also: UI5 linter clean for `webapp/ext/**`.

**Not verified:** connection to a real S/4HANA system (host is a placeholder), RAP implementation, list report multi‑select
release, creating new profiles/rules through the UI, accessibility review, the generated OPA5 skeleton tests (not adapted),
the UI5 manifest validator (the tool itself failed with a schema error). `npm audit` reports vulnerabilities in the Fiori tools
dev toolchain – not shipped code.
