# Field Order Line getBinV2 Region Default Bin Design

## Purpose

This document proposes a focused design change for the Field Order Line Install destination-bin logic.

The current integration still depends on a legacy model of customer-specific bins. That model is fragile because the bin search relies on payload fields such as customer Phytech ID, area, plot, project, and service territory. In the real data, some payloads are partial or inconsistent, so the current `getBin` logic can select a wrong legacy/default customer bin.

The proposed solution is to stop resolving installed customer inventory by customer-specific bin metadata. Instead, NetSuite should resolve the install destination from the customer region:

```text
Field Order Line customer
  -> customer region
  -> region default customer location
  -> one default installation bin for that region
```

This document also maps the current Install and Remove paths so the customer can confirm which existing flows should remain unchanged and which should use the new `getBinV2` logic.

## Scope

### In Scope

```text
Serialized Install destination resolution
  -> replace legacy getBin(rec, BinTypes.CUSTOMER)
  -> use getBinV2(rec)
  -> add stricter setup validation before creating the positive Install IA
Remove destination model review
  -> document how Tech Main and Tech Faulty bins work today
  -> decide whether removed/faulty serials still go to technician faulty bins
  -> decide whether removed/faulty serials should instead go to region-level faulty bins
```

### Out Of Scope Unless Confirmed Separately

```text
Remove / Uninstall transaction logic unless regional faulty-bin destination is approved
Non-serialized item behavior
Sales Order / Item Fulfillment business behavior
Related item business behavior
Duplicate Remove / same-bin validation fix
Purchase Order logic
```

Remove is included in this document because the customer needs to understand that Remove does not use `getBin` today. However, if the customer stops maintaining technician/customer-style bin structures and wants faulty inventory centralized by region, then Remove destination logic must also be changed.

## Current Problem

The current serialized Install path calls:

```javascript
var bin = getBin(rec, BinTypes.CUSTOMER);
```

The legacy `getBin` function searches customer bins using these Field Order Line values:

```text
custrecord_phy_fil_area_id
custrecord_phy_fil_plot_id
custrecord_phy_fil_project_id
custrecord_phy_account_phytech_id
custrecord_phy_bin_type = Customer
```

When those fields are blank, the search can match many legacy/default bins where the same metadata fields are also blank. Because the search iterates through results and keeps assigning the last returned bin, a California customer can end up installed into a Washington default bin.


## Proposed Business Model

Use one default installation bin per region instead of one bin per customer/site.

```text
Customer
  -> customer cseg_region
  -> region default customer location
  -> bin at that location with type = Region Default Bin
  -> serialized installed inventory is added there
```


## New Bin Type

Add a new value to the existing bin type list.

| Bin type | Current/new | Purpose |
| --- | --- | --- |
| Customer | Existing | Legacy per-customer/per-site bin model. |
| Tech Main | Existing | Technician working stock. |
| Tech Faulty | Existing | Removed/faulty inventory destination. |
| Region Default Bin | New | One default customer-installation bin per region. |

The exact internal id of `Region Default Bin` must be confirmed after the NetSuite list value is created.

## Current Technician And Faulty Bin Model

The current helper uses three bin type values:

```text
Customer = 1
Tech Main = 2
Tech Faulty = 3
```

These are NetSuite bins filtered by the custom bin field `custrecord_phy_bin_type`.

### Tech Main Bin Usage

```text
Tech Main
  -> technician working-stock bin
  -> used mainly by Install source/availability logic
  -> non-serialized Install checks item availability in the technician location and Tech Main bin
```

### Tech Faulty Bin Usage

```text
Tech Faulty
  -> removed/faulty inventory destination bin
  -> used by Remove / Box Remove
  -> found under the Remove destination location, not under the customer/source location
```

Current Remove destination resolution:

```text
Remove with technician
  -> technician location
  -> Tech Faulty bin at that technician location

Remove without technician
  -> region Generic MRB location
  -> Tech Faulty bin at the Generic MRB location

If region Generic MRB is missing
  -> subsidiary default location
  -> Tech Faulty bin at the subsidiary default location
```

This means the current setup expects faulty bins to exist at every location that can become a Remove destination. If the future design stops maintaining technician-level faulty bins, then Remove cannot stay fully unchanged.

### Future Faulty-Bin Decision

TO BE DECIDED: Should removed/faulty serialized inventory continue moving to technician/fallback `Tech Faulty` bins, or should Remove use a region-level faulty destination?

Possible future model:

```text
Remove serial
  -> find serial current location
  -> resolve customer/FIL region
  -> resolve region default faulty location
  -> resolve one faulty bin at that region location
  -> transfer serial to region faulty destination
```

If this model is approved, each active region must have a configured faulty destination location and bin. This may be the same physical location as the region default customer location, or a separate MRB/faulty location, depending on the phytec's operations.

## Required Setup And Validation

The new logic should be stricter than the current process. If required setup data is missing, the script should fail clearly before creating the positive Install IA.

| Requirement | Reason |
| --- | --- |
| Every active customer must have a Phytech ID | Required for reliable integration identity and traceability. |
| Every active customer must have `cseg_region` | `getBinV2` resolves the install destination from the customer region. |
| Every region must have a default customer location | This is where installed customer inventory will be held for that region. |
| Every region default customer location must have exactly one `Region Default Bin` | This is the destination bin for serialized installs in that region. |

Recommended validation behavior:

```text
Missing customer
  -> fail with clear FIL_CUSTOMER_MISSING error

Customer without Phytech ID
  -> fail with clear CUSTOMER_PHYTECH_ID_MISSING error

Customer without region
  -> fail with clear CUSTOMER_REGION_MISSING error

Region without default customer location
  -> fail with clear REGION_DEFAULT_LOCATION_MISSING error

Default customer location without Region Default Bin
  -> fail with clear REGION_DEFAULT_BIN_MISSING error
```

## getBinV2 Design

### Responsibility

`getBinV2` should answer one question only:

```text
For this Field Order Line customer, what region default bin/location should receive installed inventory?
```

It should not create a per-customer bin. It should not search by plot, project, area, or service territory.

### Resolution Chain

```text
Field Order Line
  -> custrecord_phy_fil_account
     -> customer record
        -> validate customer Phytech ID
        -> validate customer cseg_region
           -> resolve region default customer location
              -> find bin at that location with type = Region Default Bin
                 -> return { binNumber, location }
```

### Region Source

The FIL already has `cseg_region`, populated in `beforeSubmit` from the selected customer. For safety, `getBinV2` should still validate the customer directly.

Recommended logic:

```text
1. Read FIL customer.
2. Lookup customer Phytech ID and customer cseg_region.
3. If customer cseg_region is missing, fail.
4. If FIL cseg_region is empty, use customer cseg_region.
5. If FIL cseg_region exists but does not match customer cseg_region, fail with a mismatch error.
```

## Current Flow Summary

The User Event currently runs this high-level flow:

```text
External payload
  -> customrecord_fhy_fil
  -> beforeSubmit sets cseg_region from customer
  -> afterSubmit checks status New or Resubmit
  -> FieldOrderLineInstallation(rec)
     -> resolve item and isSerial
     -> normalize Box Install / Box Remove
     -> route to Install or Remove branch
```

## Install Paths And Required Changes

### Install Common Steps

All Install paths start with the same core logic:

```text
Install / Box Install
  -> resolve item and isSerial
  -> resolve technician/source location
  -> validate item availability or serial location
  -> optionally search Sales Order
  -> resolve subsidiary and adjustment account
  -> resolve related items
```

The proposed change does not affect these common steps. The change happens only when serialized Install needs the destination bin for the positive Install IA.

### Path I1: Serialized Install, Maintenance Or No Sales Order

This is the path that matches `FIL1365924` and the WA Customers warehouse issue.

```text
Conditions
  -> action = Install or Box Install
  -> isSerial = true
  -> order type = Maintenance, or no Sales Order is found
  -> remove IA flag = false
  -> install IA flag = false

Current behavior
  -> find serial current location
  -> create negative Remove IA from source location
  -> call legacy getBin(rec, CUSTOMER)
  -> create positive Install IA into returned bin/location
  -> update serial installation date

Proposed behavior
  -> find serial current location: unchanged
  -> create negative Remove IA: unchanged
  -> call getBinV2(rec)
  -> create positive Install IA into region default bin/location
  -> update serial installation date using region default location
```

Change required: Yes.

This is the primary path to fix.

### Path I2: Serialized Install, Installation/Inventory Order, Sales Order Found

```text
Conditions
  -> action = Install or Box Install
  -> isSerial = true
  -> order type = Installation or Inventory
  -> Sales Order found by opportunity id and item
  -> IF flag = false

Current behavior
  -> create Item Fulfillment from Sales Order
  -> mark IF created
  -> resolve related items
  -> if related items exist, create Remove IA for related/source-side items
  -> call legacy getBin(rec, CUSTOMER)
  -> create positive Install IA into returned bin/location
  -> update serial installation date

Proposed behavior if current business flow stays
  -> Item Fulfillment stays unchanged
  -> related items stay unchanged
  -> call getBinV2(rec)
  -> positive Install IA goes to region default bin/location
```

Change required: Yes, if this path remains active.

TO BE DECIDED: Should the Sales Order / Item Fulfillment path remain valid, or should all installs use the same IA pair logic regardless of Sales Order?

### Path I3: Serialized Install Resubmit, Remove IA Already Created But Install IA Missing

```text
Conditions
  -> action = Install or Box Install
  -> isSerial = true
  -> remove IA flag = true
  -> install IA flag = false

Current behavior
  -> skip serial source-location search
  -> skip Remove IA creation
  -> call legacy getBin(rec, CUSTOMER)
  -> create positive Install IA

Proposed behavior
  -> skip already-created source transaction: unchanged
  -> call getBinV2(rec)
  -> create positive Install IA into region default bin/location
```

Change required: Yes.


### Path I4: Serialized Install Resubmit, Install IA Already Created

```text
Conditions
  -> action = Install or Box Install
  -> isSerial = true
  -> install IA flag = true

Current behavior
  -> code still resolves destination bin before checking whether install IA is already created
  -> skip Install IA creation

Proposed behavior
  -> no transaction should be created
  -> recommended small improvement: if install IA flag is true, skip destination-bin resolution entirely
```

Change required for the destination resolver: Yes if the current structure remains. Recommended improvement: add an early skip when install IA already exists, so completed resubmits do not fail only because destination setup is now stricter.


### Path I5: Non-Serialized Install With Sales Order

```text
Conditions
  -> action = Install or Box Install
  -> isSerial = false
  -> Sales Order found

Current behavior
  -> validate stock in technician main bin
  -> create Item Fulfillment
  -> resolve related items
  -> maybe create source-side Remove IA for related items
  -> skip customer Install IA because item is not serialized
  -> no getBin call

Proposed behavior
  -> unchanged under the minimum-change design
```

Change required: No.

TO BE DECIDED: Should non-serialized related items still be handled this way, or should any non-serialized installed stock also be placed into the region default bin?

### Path I6: Non-Serialized Install Without Sales Order

```text
Conditions
  -> action = Install or Box Install
  -> isSerial = false
  -> no Sales Order found

Current behavior
  -> validate stock in technician main bin
  -> create negative Remove IA from technician/source location
  -> skip customer Install IA because item is not serialized
  -> no getBin call

Proposed behavior
  -> unchanged under the minimum-change design
```

Change required: No.

TO BE DECIDED: Is this intentional? Today this path removes non-serialized inventory from the technician/source side but does not add it into a customer or region default location. is that a possible path or it can be ignored?

### Install Path Summary

| Path | Uses legacy getBin today | Change with getBinV2 | Notes |
| --- | --- | --- | --- |
| Serialized Install, Maintenance/no SO | Yes | Yes | Main incident path. |
| Serialized Install, SO found | Yes | Yes, if flow remains | Requires customer confirmation about SO/IF behavior. |
| Serialized Install partial resubmit | Yes | Yes | Important for retry safety. |
| Serialized Install already completed | Yes in current structure | Prefer early skip | Avoids unnecessary bin resolution. |
| Non-serialized Install with SO | No | No | TO BE DECIDED |
| Non-serialized Install without SO | No | No | TO BE DECIDED |

## Remove / Uninstall Paths And Required Changes

Remove does not call legacy `getBin` at all.

Current Remove destination is resolved through technician/faulty-bin logic:

```text
Remove / Box Remove
  -> if technician exists, resolve technician location
  -> if technician is blank, resolve region Generic MRB location
  -> if Generic MRB is missing, resolve subsidiary default location
  -> find Tech Faulty bin at the chosen destination location
  -> find serial current location
  -> create Inventory Transfer from serial location to the faulty destination
```

Therefore, `getBinV2` does not directly modify Remove logic. But the broader bin-model change does affect the Remove design decision. If faulty bins will no longer exist under technician locations, Remove must be changed to use a region-level faulty destination.

### Path R1: Serialized Remove With Technician And Serial Location Found

```text
Conditions
  -> action = Remove or Box Remove
  -> isSerial = true
  -> technician is populated
  -> technician location found
  -> technician faulty bin found
  -> serial current location found

Current behavior
  -> create Inventory Transfer from serial current location to technician location
  -> destination bin = technician faulty bin

Proposed behavior
  -> unchanged only if technician faulty bins remain part of the future setup
  -> if technician faulty bins are retired, use region-level faulty destination instead
```

Change required for getBinV2 only: No.

Change required if faulty inventory is centralized by region: Yes.

When a serial was installed using `getBinV2`, this path should still work because it dynamically finds the serial's current location. The source will simply be the region default customer location instead of a legacy customer bin location.

TO BE DECIDED: Should Remove with technician still send faulty inventory to the technician's `Tech Faulty` bin, or should it ignore technician destination and send faulty inventory to the region faulty destination?

### Path R2: Serialized Remove Without Technician

```text
Conditions
  -> action = Remove or Box Remove
  -> isSerial = true
  -> technician is blank
  -> region fallback location is found
  -> faulty bin exists at fallback location
  -> serial current location found

Current behavior
  -> use region generic MRB, or subsidiary default location fallback
  -> create Inventory Transfer from serial current location to fallback/faulty destination

Proposed behavior
  -> unchanged if Generic MRB / subsidiary fallback remains the approved no-technician destination
  -> otherwise resolve the same region-level faulty destination used by all Removes
```

Change required for getBinV2 only: No.

Change required if faulty inventory is centralized by region: Yes.

TO BE DECIDED: Should technician-missing Remove continue using Generic MRB / subsidiary default fallback, or should it use a region-specific faulty/default location?

### Path R3: Serialized Remove, Serial Not Found

```text
Conditions
  -> action = Remove or Box Remove
  -> isSerial = true
  -> serial cannot be found on hand by item/serial search

Current behavior
  -> findLocationForItemWithSerial throws an error
  -> FIL is marked Failed
  -> fallback IA branch in code is normally unreachable because the lookup throws first

Proposed behavior
  -> unchanged for getBinV2 scope
```

Change required for getBinV2 only: No.


### Path R4: Serialized Remove Duplicate / Same Destination

```text
Conditions
  -> action = Remove or Box Remove
  -> isSerial = true
  -> serial is already in the technician/faulty destination

Current behavior
  -> code attempts Inventory Transfer anyway
  -> NetSuite may reject with: The from and to bins must be different
  -> FIL is marked Failed with a generic NetSuite error

Proposed behavior
  -> detect before creating the Inventory Transfer that the serial is already in the target faulty destination
  -> fail the FIL with a clear duplicate Remove error
  -> do not create a new transaction
```

Change required for getBinV2 only: No.

Recommended separate fix: Add a Remove duplicate validation guard. If the serial is already in the target faulty destination, stop processing and fail the FIL with a clear error such as `DUPLICATE_REMOVE_SERIAL_ALREADY_IN_TARGET`.

### Path R5: Non-Serialized Remove

```text
Conditions
  -> action = Remove or Box Remove
  -> isSerial = false

Current behavior
  -> resolve technician/fallback location
  -> resolve technician faulty bin
  -> no transaction is created
  -> FIL can be marked Completed

Proposed behavior
  -> unchanged under getBinV2 scope
```

Change required: No.

Customer question: Is it intentional that non-serialized Remove creates no movement transaction?

### Path R6: Technician Faulty Bin Missing

```text
Conditions
  -> action = Remove or Box Remove
  -> technician/fallback location is found
  -> no TECH_FAULTY bin is found at that location

Current behavior
  -> throw CANT_FIND_TECH_BIN
  -> FIL is marked Failed

Proposed behavior
  -> fail clearly if the selected Remove destination has no faulty bin
  -> if regional faulty destination is approved, validate the region faulty bin instead of technician/fallback faulty bin
```

Change required for getBinV2 only: No.

Change required if faulty inventory is centralized by region: Yes.

TO BE DECIDED: Should missing technician/fallback faulty bins remain a setup failure, or should Remove use a required region-level faulty bin?

### Remove Path Summary

| Path | Uses legacy getBin today | Change with getBinV2 | Notes |
| --- | --- | --- | --- |
| Serialized Remove with technician | No | No direct getBinV2 change | TO BE DECIDED: keep technician faulty bin or use region faulty destination. |
| Serialized Remove without technician | No | No direct getBinV2 change | TO BE DECIDED: keep Generic MRB/subsidiary fallback or use region faulty destination. |
| Serialized Remove serial not found | No | No | - |
| Serialized duplicate Remove / same-bin | No | No direct getBinV2 change | Separate clear duplicate Remove validation recommended. |
| Non-serialized Remove | No | No | Existing no-transaction behavior. |
| Missing faulty bin | No | No direct getBinV2 change | Destination setup depends on future faulty-bin model. |

## Full Change Matrix

| Area | Current behavior | Proposed getBinV2 impact |
| --- | --- | --- |
| Serialized Install destination | Legacy customer bin by plot/project/area/account | Change to region default bin. |
| Serialized Install source removal | Negative IA from source location | No change. |
| Serialized Install positive IA | Positive IA into returned bin/location | Same transaction, new destination. |
| Install serial installation date | Updated after positive IA | No change, but location is region default location. |
| Sales Order / Item Fulfillment | Existing IF path when SO found | Pending business confirmation. |
| Purchase Order | No inspected PO path in Field Order Line helper | No change unless business identifies a separate PO requirement. |
| Related items | Existing related-item search and movement | Pending business confirmation. |
| Non-serialized Install | No getBin call | No change. |
| Remove / Uninstall source lookup | Finds serial current location and transfers it out | No direct getBinV2 change; validate saved search finds serials in Region Default Bins. |
| Remove faulty destination | Technician location, region Generic MRB, or subsidiary default location with Tech Faulty bin | TO BE DECIDED: keep current model or move all faulty inventory to region-level faulty destination. |
| Duplicate Remove / same destination | Attempts Inventory Transfer and may fail with generic NetSuite same-bin error | Separate fix recommended: fail clearly before creating transaction. |
| Legacy customer bins | Primary destination today | Do not use as fallback for this new Install destination logic. |


## Customer-Facing Summary

The proposed fix is to stop resolving installed customer inventory by plot/project/area/customer-bin metadata. Instead, NetSuite will resolve the destination from the customer region. Each region will have one default customer location and one default installation bin. All serialized installed inventory for customers in that region will be added to that bin.

This is a minimum-change design for serialized Install destination resolution. The source-side Install logic remains the same, including serial lookup, negative Inventory Adjustment, cost capture, related items, status updates, and resubmit flags.

Remove/Uninstall does not call legacy `getBin`, but the new regional inventory model creates an important customer decision. Today, Remove transfers removed serials to a `Tech Faulty` bin under the technician location, region Generic MRB location, or subsidiary default location. If the customer no longer wants faulty bins under those locations, Remove should be changed to use a region-level faulty destination instead.

The only direct functional change is the destination resolver used by serialized Install before creating the positive Install Inventory Adjustment.
