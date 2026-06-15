# Field Order Line getBinV2 Region Default Bin Design

## Purpose

This document proposes a minimum-change design for fixing the Field Order Line Install destination-bin logic.

\*\*short dectription of the problm and solution" and scope  

## Current Problem

Currently the sycnt between phytec and netustie largly depnds on on 1 customer type bin per each customer. with many complicated fallbacks using other data passed from the intial phytec paylod such as thenication custom bin and so on.  another problm that will need to be solved is to standdize the paylod that is passed by phtec as we seen many examples of inconsits partial paylod with missing critical data.

part of the solution is also implemt stricter ERrror on mandatory fields wic be dicused later in the design.

The current serialized Install path calls:


```javascript
var bin = getBin(rec, BinTypes.CUSTOMER);
```

The legacy `getBin` function searches customer bins using these fields from the Field Order Line:


```text
custrecord_phy_fil_area_idcustrecord_phy_fil_plot_idcustrecord_phy_fil_project_idcustrecord_phy_account_phytech_idcustrecord_phy_bin_type = Customer
```

When these values are blank, the search can match many legacy/default bins where those same metadata fields are also blank. In the reported case, the search result iteration ended with `WA:Customers-Default`, so a California customer serial was installed into the Washington customers warehouse.

The business direction is to stop using one bin per customer/plot/project. The new model should use one default installation bin per region.

## Proposed Business Model

Use one default customer-installation bin per region.


```text
Customer  -> customer cseg_region  -> region default customer location  -> one bin at that location with type = Region Default Bin  -> serialized installed inventory is added there
```

Example target result:


```text
Customer: CMC FARMING - DELANOCustomer region: CaliforniaRegion default customer location: CA Customers warehouseRegion default bin: CA:Customers-DefaultInstall positive IA destination: CA Customers warehouse / CA:Customers-Default
```

## Proposed New Bin Type

Add a new value to the existing bin type list.

| Bin type | Current/new | Purpose |
| --- | --- | --- |
| Customer | Existing | Legacy per-customer/per-site bin model |
| Tech Main | Existing | Technician working stock |
| Tech Faulty | Existing | Removed/faulty inventory destination |
| Region Default Bin | New | One default customer-installation bin per region |

The exact internal id of `Region Default Bin` must be confirmed after setup. The code should not assume `4` unless that is the actual configured value in the account.

## Required Setup

Before deploying the new logic, the account setup should be aligned:

| Requirement | Reason |
| --- | --- |
| Every active customer must have a Phytech ID | Required for reliable integration identity and traceability. This also prevents ambiguous customer mapping. |
| Every active customer must have `cseg_region` | `getBinV2` resolves the install destination from the customer region. |
| Every region must have one default customer location | This is the location where installed customer inventory will be held for that region. |
| Every region default customer location must have exactly one `Region Default Bin` | This is the destination bin for serialized installs in that region. |
| Legacy customer bins can remain during transition | They are only used as guarded fallback, not as the primary path. |

Recommended validation rule:


```text
Customer without Phytech ID or region should fail with a clear setup error before inventory movement.
```

## getBinV2 Design

### Responsibility

`getBinV2` should only answer one question:


```text
For this Field Order Line customer, what region default bin/location should receive installed inventory?
```

It should not create a per-customer bin. It should not search by plot, project, area, or service territory.

### Resolution Chain


```text
Field Order Line  -> custrecord_phy_fil_account     -> customer record        -> validate customer Phytech ID        -> customer cseg_region           -> region default customer location              -> bin at that location with type = Region Default Bin                 -> return { binNumber, location }
```

### Preferred Data Source For Region

The FIL already has `cseg_region`, populated in `beforeSubmit` from the selected customer. For safety, `getBinV2` should still validate the customer directly:


```text
1. Read FIL customer.2. Lookup customer Phytech ID and customer cseg_region.3. If FIL cseg_region is empty, use customer cseg_region.4. If both are present but different, fail with a clear mismatch error.
```

This avoids using a stale or incorrect FIL region if the customer setup changed.

### Region Default Location

The cleanest setup is a direct field on the Region record, for example:


```text
Region -> Default Customer Location
```

If the account already has a different approved source for region default customer locations, the implementation can use it. The important rule is that the result must be region-specific and deterministic.

Possible options to confirm:

| Option | Description | Recommendation |
| --- | --- | --- |
| Direct region field | `customrecord_cseg_region` has a default customer location field | Preferred. Most explicit and easiest to audit. |
| Location search by region/type | Search `location` by `cseg_region` and customer-location type | Acceptable only if exactly one result is guaranteed. |
| Region -> subsidiary -> subsidiary default location | Existing fallback pattern in current code | Less precise if subsidiary has several regions or customer warehouses. |

### Pseudocode


```javascript
function getBinV2(rec) {    var customerId = rec.getValue({ fieldId: 'custrecord_phy_fil_account' });    if (common.isNullOrEmpty(customerId)) {        throw error.create({            name: 'FIL_CUSTOMER_MISSING',            message: 'Field Order Line has no customer. Cannot resolve region default bin.',            notifyOff: true        });    }    var customerFields = search.lookupFields({        type: search.Type.CUSTOMER,        id: customerId,        columns: ['cseg_region', 'custentity_or_existing_phytech_id_field']    });    var phytechId = getLookupValue(customerFields.custentity_or_existing_phytech_id_field);    if (common.isNullOrEmpty(phytechId)) {        throw error.create({            name: 'CUSTOMER_PHYTECH_ID_MISSING',            message: 'Customer has no Phytech ID. Cannot process Field Order Line safely.',            notifyOff: true        });    }    var customerRegion = getLookupValue(customerFields.cseg_region);    var filRegion = rec.getValue({ fieldId: 'cseg_region' });    var region = customerRegion || filRegion;    if (common.isNullOrEmpty(region)) {        throw error.create({            name: 'CUSTOMER_REGION_MISSING',            message: 'Customer has no region. Cannot resolve region default bin.',            notifyOff: true        });    }    if (!common.isNullOrEmpty(filRegion) && String(filRegion) != String(region)) {        throw error.create({            name: 'FIL_CUSTOMER_REGION_MISMATCH',            message: 'Field Order Line region does not match customer region.',            notifyOff: true        });    }    var locationId = getRegionDefaultCustomerLocation(region);    if (common.isNullOrEmpty(locationId)) {        throw error.create({            name: 'REGION_DEFAULT_LOCATION_MISSING',            message: 'No default customer location is configured for region ' + region,            notifyOff: true        });    }    var bin = findRegionDefaultBin(locationId);    if (common.isNullOrEmpty(bin)) {        throw error.create({            name: 'REGION_DEFAULT_BIN_MISSING',            message: 'No Region Default Bin exists at location ' + locationId,            notifyOff: true        });    }    return {        binNumber: bin.binNumber,        location: locationId    };}
```

## Legacy Fallback Policy

The customer requested `getBinV2` as the main path and legacy `getBin` as fallback.

Recommended fallback rule:


```text
Use legacy getBin only when the legacy matching fields are populated enough to avoid the original blank-field bug.
```

Do not blindly fall back to legacy `getBin` when these fields are blank:


```text
custrecord_phy_fil_area_idcustrecord_phy_fil_plot_idcustrecord_phy_fil_project_idcustrecord_phy_account_phytech_id
```

If all or most of these are blank, fallback to legacy `getBin` can reproduce the exact WA Customers warehouse issue.

Recommended transition behavior:


```text
getBinV2 succeeds  -> use region default bingetBinV2 fails because setup is missing  -> if approved by business and legacy keys are populated, call legacy getBin  -> otherwise fail with a clear setup error
```

Possible code shape:


```javascript
var bin;try {    bin = getBinV2(rec);} catch (v2Error) {    logger.error({ title: 'getBinV2 failed', details: JSON.stringify(v2Error) });    if (canUseLegacyCustomerBinFallback(rec)) {        bin = getBin(rec, BinTypes.CUSTOMER);    } else {        throw v2Error;    }}
```

## Minimum Code Change Boundary

There is one direct legacy destination-bin call in the main helper:


```javascript
var bin = getBin(rec, BinTypes.CUSTOMER);
```

It is inside the serialized Install path, after the source-side Remove IA and before the customer-side Install IA.

Minimum replacement:


```javascript
var bin = getBinV2(rec);
```

Transitional replacement:


```javascript
var bin = getBinForInstallDestination(rec);
```

Where `getBinForInstallDestination` runs `getBinV2` first and calls legacy `getBin` only under the guarded fallback policy.

## Current Flow Summary

The User Event does this:


```text
External payload  -> customrecord_fhy_fil  -> beforeSubmit sets cseg_region from customer  -> afterSubmit checks status New or Resubmit  -> FieldOrderLineInstallation(rec)     -> resolve item and isSerial     -> normalize Box Install/Box Remove     -> Install branch or Remove branch
```

The inspected Field Order Line helper has a Sales Order / Item Fulfillment path. It does not contain a Purchase Order path for this Field Order Line install/remove processing. If Purchase Orders are expected to participate in this business flow, that should be confirmed separately and treated as a different enhancement.

## Install Paths And Required Changes

### Install Common Steps

All Install paths start with this logic:


```text
Install / Box Install  -> resolve item and isSerial  -> resolve technician/source location  -> validate item availability or serial location  -> optionally search Sales Order  -> resolve subsidiary and adjustment account  -> resolve related items
```

The proposed change does not affect these common steps.

### Path I1: Serialized Install, Maintenance Or No Sales Order

This is the path that matches the reported `FIL1365924` style issue.


```text
Conditions:  -> action = Install or Box Install  -> isSerial = true  -> order type = Maintenance, or no Sales Order is found  -> remove IA flag = false  -> install IA flag = falseCurrent behavior:  -> find serial current location  -> create negative Remove IA from source location  -> call legacy getBin(rec, CUSTOMER)  -> create positive Install IA into returned bin/location  -> update serial installation dateProposed behavior:  -> find serial current location: unchanged  -> create negative Remove IA: unchanged  -> call getBinV2 as destination resolver  -> create positive Install IA into region default bin/location  -> update serial installation date using region default location
```

Change required: Yes.

This is the primary path to fix.

### Path I2: Serialized Install, Installation/Inventory Order, Sales Order Found


```text
Conditions:  -> action = Install or Box Install  -> isSerial = true  -> order type = Installation or Inventory  -> Sales Order found by opportunity id and item  -> IF flag = falseCurrent behavior:  -> create Item Fulfillment from Sales Order  -> mark IF created  -> resolve related items  -> if related items exist, create Remove IA for related/source-side items  -> call legacy getBin(rec, CUSTOMER)  -> create positive Install IA into returned bin/location  -> update serial installation dateProposed behavior if current business flow stays:  -> Item Fulfillment stays unchanged  -> related items stay unchanged  -> replace destination resolver with getBinV2  -> positive Install IA goes to region default bin/location
```

Change required: Yes, if this path remains active.

Customer question: Should the Sales Order / Item Fulfillment path remain valid, or should all installs use the same IA pair logic regardless of Sales Order?

### Path I3: Serialized Install Resubmit, Remove IA Already Created But Install IA Missing


```text
Conditions:  -> action = Install or Box Install  -> isSerial = true  -> remove IA flag = true  -> install IA flag = falseCurrent behavior:  -> skip serial source-location search  -> skip Remove IA creation  -> call legacy getBin(rec, CUSTOMER)  -> create positive Install IAProposed behavior:  -> skip already-created source transaction: unchanged  -> call getBinV2  -> create positive Install IA into region default bin/location
```

Change required: Yes.

This path matters because failed records may be resubmitted after only the negative IA succeeded.

### Path I4: Serialized Install Resubmit, Install IA Already Created


```text
Conditions:  -> action = Install or Box Install  -> isSerial = true  -> install IA flag = trueCurrent behavior:  -> code still resolves destination bin before checking whether install IA is already created  -> skip Install IA creationProposed behavior:  -> no transaction should be created  -> recommended improvement: if install IA flag is true, skip destination-bin resolution entirely
```

Change required for getBinV2: Technically yes if the current structure remains, because the code still calls the bin resolver. Recommended small improvement: early skip when install IA already exists.

Customer question: Is it acceptable to add this small guard to reduce unnecessary errors on already-completed resubmits?

### Path I5: Non-Serialized Install With Sales Order


```text
Conditions:  -> action = Install or Box Install  -> isSerial = false  -> Sales Order foundCurrent behavior:  -> validate stock in technician main bin  -> create Item Fulfillment  -> resolve related items  -> maybe create source-side Remove IA for related items  -> skip customer Install IA because item is not serialized  -> no getBin call
```

Change required: No.

Customer question: Should non-serialized related items still be handled this way, or should any non-serialized installed stock also be placed into the region default bin?

### Path I6: Non-Serialized Install Without Sales Order


```text
Conditions:  -> action = Install or Box Install  -> isSerial = false  -> no Sales Order foundCurrent behavior:  -> validate stock in technician main bin  -> create negative Remove IA from technician/source location  -> skip customer Install IA because item is not serialized  -> no getBin call
```

Change required: No under the minimum-change design.

Customer question: Is this intentional? Today this path removes non-serialized inventory from the technician/source side but does not add it into a customer or region default location. If non-serialized items are consumables, that may be correct. If they should remain on customer inventory, this is a separate enhancement.

### Install Path Summary

| Path | Uses legacy getBin today | Change with getBinV2 | Notes |
| --- | --- | --- | --- |
| Serialized Install, Maintenance/no SO | Yes | Yes | Main incident path. |
| Serialized Install, SO found | Yes | Yes, if flow remains | Requires customer confirmation about SO/IF behavior. |
| Serialized Install partial resubmit | Yes | Yes | Important for retry safety. |
| Serialized Install already completed | Yes in current structure | Prefer early skip | Avoids unnecessary bin resolution. |
| Non-serialized Install with SO | No | No | Confirm business expectation only. |
| Non-serialized Install without SO | No | No | Existing behavior may be intentional consumable flow. |

## Remove / Uninstall Paths And Required Changes

Remove does not call legacy `getBin` at all.

Remove destination is resolved through technician/faulty-bin logic:


```text
Remove / Box Remove  -> resolve technician location, or region fallback if technician is blank  -> getTechnicianBin(techLocation, TECH_FAULTY, true)  -> find serial current location  -> create Inventory Transfer from serial location to technician/faulty destination
```

Therefore, the `getBinV2` change does not directly modify Remove logic.

### Path R1: Serialized Remove With Technician And Serial Location Found


```text
Conditions:  -> action = Remove or Box Remove  -> isSerial = true  -> technician is populated  -> technician location found  -> technician faulty bin found  -> serial current location foundCurrent behavior:  -> create Inventory Transfer from serial current location to technician location  -> destination bin = technician faulty binProposed behavior:  -> unchanged
```

Change required: No.

When a serial was installed using `getBinV2`, this path should still work because it dynamically finds the serial’s current location. The source will simply be the region default customer location instead of a legacy customer bin location.

### Path R2: Serialized Remove Without Technician


```text
Conditions:  -> action = Remove or Box Remove  -> isSerial = true  -> technician is blank  -> region fallback location is found  -> faulty bin exists at fallback location  -> serial current location foundCurrent behavior:  -> use region generic MRB, or subsidiary default location fallback  -> create Inventory Transfer from serial current location to fallback/faulty destinationProposed behavior:  -> unchanged
```

Change required: No.

Customer question: Should technician-missing Remove continue using Generic MRB / subsidiary default fallback, or should it use a region-specific faulty/default location?

### Path R3: Serialized Remove, Serial Not Found


```text
Conditions:  -> action = Remove or Box Remove  -> isSerial = true  -> serial cannot be found on hand by item/serial searchCurrent behavior:  -> findLocationForItemWithSerial throws an error  -> FIL is marked Failed  -> fallback IA branch in code is normally unreachable because the lookup throws firstProposed behavior:  -> unchanged for getBinV2 scope
```

Change required: No.

Customer question: Should missing-serial Remove fail, or should it create a fallback adjustment? This is separate from the region default bin fix.

### Path R4: Serialized Remove Duplicate / Same Destination


```text
Conditions:  -> action = Remove or Box Remove  -> isSerial = true  -> serial is already in the technician/faulty destinationCurrent behavior:  -> code attempts Inventory Transfer anyway  -> NetSuite may reject with: The from and to bins must be different  -> FIL is marked FailedProposed behavior for getBinV2 scope:  -> unchanged
```

Change required for getBinV2: No.

Recommended separate fix: Add Remove idempotency guard so repeated Remove messages are treated as already completed when the serial is already in the target destination.

### Path R5: Non-Serialized Remove


```text
Conditions:  -> action = Remove or Box Remove  -> isSerial = falseCurrent behavior:  -> resolve technician/fallback location  -> resolve technician faulty bin  -> no transaction is created  -> FIL can be marked CompletedProposed behavior:  -> unchanged under getBinV2 scope
```

Change required: No.

Customer question: Is it intentional that non-serialized Remove creates no movement transaction?

### Path R6: Technician Faulty Bin Missing


```text
Conditions:  -> action = Remove or Box Remove  -> technician/fallback location is found  -> no TECH_FAULTY bin is found at that locationCurrent behavior:  -> throw CANT_FIND_TECH_BIN  -> FIL is marked FailedProposed behavior:  -> unchanged under getBinV2 scope
```

Change required: No.

Customer question: Should missing faulty bins remain a setup failure, or should there be a region-level fallback faulty bin?

### Remove Path Summary

| Path | Uses legacy getBin today | Change with getBinV2 | Notes |
| --- | --- | --- | --- |
| Serialized Remove with technician | No | No | Dynamically finds current serial location. |
| Serialized Remove without technician | No | No | Uses current fallback logic. |
| Serialized Remove serial not found | No | No | Separate missing-serial behavior. |
| Serialized duplicate Remove / same-bin | No | No | Separate idempotency fix recommended. |
| Non-serialized Remove | No | No | Existing no-transaction behavior. |
| Missing technician faulty bin | No | No | Setup issue, not related to getBinV2. |

## Full Change Matrix

| Area | Current behavior | Proposed getBinV2 impact |
| --- | --- | --- |
| Serialized Install destination | Legacy customer bin by plot/project/area/account | Change to region default bin |
| Serialized Install source removal | Negative IA from source location | No change |
| Serialized Install positive IA | Positive IA into returned bin/location | Same transaction, new destination |
| Install serial installation date | Updated after positive IA | No change, but location is region default location |
| Sales Order / Item Fulfillment | Existing IF path when SO found | Pending business confirmation |
| Purchase Order | No inspected PO path in Field Order Line helper | No change unless business identifies a separate PO requirement |
| Related items | Existing related-item search and movement | Pending business confirmation |
| Non-serialized Install | No getBin call | No change |
| Remove / Uninstall | Technician/faulty destination, Inventory Transfer | No change |
| Legacy customer bins | Primary destination today | Transitional guarded fallback only |

## Customer Decisions Needed

1.  Confirm the new destination model: one `Region Default Bin` per region default customer location.
2.  Confirm every customer must have a Phytech ID and `cseg_region` before FIL processing.
3.  Confirm the authoritative source for region default customer location.
4.  Confirm the new bin type name and internal id for `Region Default Bin`.
5.  Confirm whether legacy `getBin` fallback is allowed only when legacy fields are populated, or whether missing setup should fail immediately.
6.  Confirm whether Sales Order / Item Fulfillment installs should remain as-is.
7.  Confirm whether Purchase Orders are expected to participate in Field Order Line install/remove processing. The inspected helper does not currently include a PO branch.
8.  Confirm whether related items should still move with the parent item into the region default bin.
9.  Confirm whether non-serialized Install behavior is intentional.
10.  Confirm whether non-serialized Remove behavior is intentional.
11.  Confirm whether Remove duplicate/same-bin idempotency should be handled as a separate fix in the same deployment or later.

## Recommended Implementation Plan

1.  Add new bin type value `Region Default Bin`.
2.  Configure one region default customer location per region.
3.  Create one `Region Default Bin` at each region default customer location.
4.  Add `getBinV2` to `Be.Lib.Installation_FOI.js`.
5.  Add a guarded fallback wrapper, for example `getBinForInstallDestination`.
6.  Replace the serialized Install destination call with the wrapper.
7.  Add clear errors for missing customer, missing Phytech ID, missing region, missing default location, and missing region default bin.
8.  Test in Sandbox using both historical incident records and clean payloads.
9.  Deploy after customer confirms open decisions.

## Suggested Test Cases

| Test | Input | Expected result |
| --- | --- | --- |
| Historical incident replay | `FIL1365924`\-style CA customer, blank area/plot/project | Positive Install IA goes to CA region default bin, not WA. |
| Another CA serial | `FIL1365912` / serial `2479279` style data | Positive Install IA goes to CA region default bin. |
| Full legacy payload | Customer has area/plot/project/account values | `getBinV2` still wins; legacy fallback is not used. |
| Missing customer region | Customer has no `cseg_region` | FIL fails with clear customer-region setup error before destination IA. |
| Missing Phytech ID | Customer has no Phytech ID | FIL fails with clear customer setup error. |
| Missing region default bin | Region location exists but no bin type `Region Default Bin` | FIL fails clearly, or uses guarded legacy fallback only if approved. |
| Remove after getBinV2 install | Serial sits in region default bin, then Remove arrives | Remove transfers from region default location to technician faulty bin. |
| Duplicate Remove | Serial already in technician faulty bin | Existing behavior unchanged unless separate idempotency fix is included. |

## Recommended Customer-Facing Summary

The proposed fix is to stop resolving installed customer inventory by plot/project/area/customer-bin metadata. Instead, NetSuite will resolve the destination from the customer region. Each region will have one default customer location and one default installation bin. All serialized installed inventory for customers in that region will be added to that bin.

This is a minimum-change design. The source-side logic remains the same, including technician/source location resolution, serial lookup, negative Inventory Adjustment, cost capture, related items, status updates, and resubmit flags. Remove/Uninstall processing also remains the same because it does not use the customer bin logic. It simply finds the serial’s current location and transfers it to the technician/faulty destination.

The only direct functional change is the destination resolver used by serialized Install before creating the positive Install Inventory Adjustment.
