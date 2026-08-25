# Niagara N4 / IQVISION Alarm BQL Investigation – BACnet Device Communication Alarms

## 1. Executive summary

This investigation was carried out on **IQVISION 4.14.0.162** using the Niagara Workbench **BQL Query** view against the `alarm:` database.

The objective was to create a reusable BQL query that identifies BACnet devices and counts their communication/ping alarm occurrences over the **last 30 days**, without hard-coding a particular BACnet network, site, or device naming convention.

### Best-known working query

```text
alarm:|bql:select alarmData.sourceName as 'BACnet Device',count('') as 'Number of Device Communication Alarms - Last 30 Days' where period = last30Days and source like '*Drivers/BacnetNetwork*' and alarmData.msgText = '%lexicon(driver:pingSuccess)%'
```

### What this query does

It queries the Niagara alarm database and:

1. Restricts the time period to the dynamically calculated **last 30 days**.
2. Restricts the alarm source to something under a Niagara **BacnetNetwork**.
3. Restricts the alarm message to the Niagara driver ping-success lexicon:
   ```text
   %lexicon(driver:pingSuccess)%
   ```
4. Groups the matching alarm records by:
   ```text
   alarmData.sourceName
   ```
5. Counts the records using:
   ```text
   count('')
   ```
6. Displays the result as:
   - `BACnet Device`
   - `Number of Device Communication Alarms - Last 30 Days`

The Workbench Collection Table can then be sorted by clicking the count column heading, so BQL `ORDER BY` is unnecessary.

### Important interpretation

The count should be understood as the number of **historical device communication/ping alarm records represented by the matching recovered (`pingSuccess`) alarm records**.

Do **not** automatically describe the count as the exact number of times a BACnet device went offline without considering the alarm configuration and record semantics. In this investigation, the evidence strongly indicates that a ping alarm record is created when communication fails and subsequently contains the `pingSuccess` recovery message when the alarm returns to normal.

---

# 2. Environment and original problem

### Niagara/IQVISION version

```text
IQVISION 4.14.0.162
```

### Original problem

BACnet networks over IP were generally healthy, but one BACnet network, **1183**, had approximately 9 of 25 energy meters repeatedly going online/offline.

The immediate diagnostic requirement was to determine how frequently each BACnet device had generated communication alarms over the preceding 30 days.

The original D2-183 BACnet network contained devices such as:

```text
D2-183 D2 - Basement Extract Plant (XB2)
D2-183 D2 - Compactor
D2-183 D2 - D2/3 DB
D2-183 D2 - DB D2/6
D2-183 D2 - DB D2G/5
D2-183 D2 - DB/D2/G/9 Supply
D2-183 D2 - Goods Lift G2
...
```

Some BACnet devices were named `BacnetDevice`, `BacnetDevice1`, etc., while others had meaningful custom slot names.

This difference became important later.

---

# 3. Initial alarm database investigation

The first useful working query was:

```text
alarm:|bql:select timestamp,sourceName,sourceState,normalTime
```

`sourceName` was initially `NULL`.

The following worked:

```text
alarm:|bql:select timestamp,alarmData.sourceName,sourceState,normalTime
```

This established that the alarm source name needed to be accessed through:

```text
alarmData.sourceName
```

rather than the top-level `sourceName`.

---

# 4. Filtering the alarm database by time

The following form did **not** work:

```text
alarm:|bql:select timestamp,alarmData.sourceName,sourceState,normalTime where timestamp in bqltime.last30days
```

A fixed absolute time did work:

```text
alarm:|bql:select timestamp,alarmData.sourceName,sourceState,normalTime where timestamp >= AbsTime '2026-07-26T00:00:00.000+01:00'
```

Filtering by source name also worked:

```text
alarm:|bql:select timestamp,alarmData.sourceName,sourceState,normalTime where timestamp >= AbsTime '2026-07-26T00:00:00.000+01:00' and alarmData.sourceName like 'D2-183*'
```

This returned approximately 1,206/1,207 records during the investigation as the alarm database changed.

---

# 5. Discovering the correct count syntax

A normal-looking aggregate query using:

```text
count(uuid)
```

did not work as required.

The following worked:

```text
alarm:|bql:select count('')
```

and returned the total number of alarm records.

With a time restriction:

```text
alarm:|bql:select count('') where timestamp >= AbsTime '2026-07-26T00:00:00.000+01:00'
```

returned the count for that period.

The key discovery was that this syntax successfully produced grouped counts:

```text
alarm:|bql:select alarmData.sourceName,count('') where timestamp >= AbsTime '2026-07-26T00:00:00.000+01:00' and alarmData.sourceName like 'D2-183*'
```

This established:

```text
count('')
```

as the reliable aggregation syntax for this Niagara alarm BQL environment.

---

# 6. Dynamic last-30-days period

A major improvement was discovered by testing:

```text
where period = last30Days
```

This works in the IQVISION 4.14 alarm BQL environment.

For example:

```text
alarm:|bql:select alarmData.sourceName,count('') where period = last30Days
```

The following approach did **not** work:

```text
alarm:?period=last30Days|bql:...
```

It returned:

```text
Target Not Found
```

Therefore, for this environment, the known-good dynamic time filter is:

```text
where period = last30Days
```

not an ORD query parameter.

This is important for future BQL query generation.

---

# 7. Understanding the alarm source

The alarm record contains a `source` field.

A test query:

```text
alarm:|bql:select source,alarmData.sourceName,count('') where timestamp >= AbsTime '2026-07-26T00:00:00.000+01:00'
```

showed that `source` contains the actual Niagara source ORD.

Examples included:

```text
local:|station:|slot:/Drivers/BacnetNetwork/D2$2d183/D2$20$2d$20Compactor
```

and:

```text
local:|station:|slot:/Drivers/BacnetNetwork/LGF$2dWC$2d133/BacnetDevice
```

This is more useful than relying only on `alarmData.sourceName`.

---

# 8. Important discovery: do not assume BACnet devices are named BacnetDevice

An initial attempt used:

```text
source like '*Drivers/BacnetNetwork/*/BacnetDevice*'
```

This successfully found devices such as:

```text
.../BacnetNetwork/LGF-WC2-133/BacnetDevice
.../BacnetNetwork/LGF-WC2-133/BacnetDevice1
.../BacnetNetwork/LGF-WC2-133/BacnetDevice2
.../BacnetNetwork/LGF-WC2-133/BacnetDevice3
```

However, it failed to find the D2-183 devices.

The reason was subsequently established.

D2 devices had custom slot names such as:

```text
.../BacnetNetwork/D2-183/D2 - Compactor
.../BacnetNetwork/D2-183/D2 - DB D2/6
.../BacnetNetwork/D2-183/D2 - Goods Lift G2
```

Therefore:

```text
source like '*Drivers/BacnetNetwork/*/BacnetDevice*'
```

does **not** mean:

> source is a `bacnet:BacnetDevice`

It actually means:

> the source path contains a component whose name begins with `BacnetDevice`.

This is a critical distinction.

### General rule

**Never use a slot-name convention as proof of Niagara component type.**

A `bacnet:BacnetDevice` can have a custom slot name.

---

# 9. Checking actual BACnet device components

The following query successfully queried the live station:

```text
station:|slot:/Drivers|bql:select name,displayName,deviceId from bacnet:BacnetDevice
```

It returned approximately 270 BACnet devices.

Example results showed:

```text
Name:
B6$2d...meter$201

Display Name:
B6 - 2nd meter 1

Device Id:
NULL
```

This proved that:

- BQL can query the actual component type `bacnet:BacnetDevice`.
- `displayName` exists on the live BACnet device component.
- `deviceId` was `NULL` in this particular query/result.
- `name` is the Niagara slot name.
- `displayName` is the configured human-readable device name.

A further test:

```text
station:|slot:/Drivers|bql:select ord,name,displayName from bacnet:BacnetDevice
```

returned `NULL` for `ord`.

Therefore `ord` was not useful as a BQL property for this purpose in this environment.

---

# 10. `source.name` does not dereference the alarm source

This query:

```text
alarm:|bql:select source,source.name,alarmData.sourceName,count('') where period = last30Days and source like '*Drivers/BacnetNetwork/*/BacnetDevice*'
```

worked syntactically, but `source.name` returned `NULL`.

This demonstrates that the alarm database's `source` value should not be assumed to behave as a live component object that can simply be dereferenced using:

```text
source.name
source.displayName
```

The safe approach is to treat:

```text
source
```

as the source ORD/reference stored in the alarm record, and:

```text
alarmData.sourceName
```

as the human-readable name captured in the alarm record.

---

# 11. Determining what the alarm message represents

The alarm records showed:

```text
%lexicon(driver:pingSuccess)%
```

for the majority of historical BACnet communication alarm records.

There were also records containing:

```text
%lexicon(driver:pingFail)%
```

A query specifically filtering Ping Fail:

```text
alarm:|bql:select source as 'BACnet Device',alarmData.sourceName as 'Display Name',count('') as 'Number of Ping Fail Alarms over last 30 days' where period = last30Days and source like '*Drivers/BacnetNetwork*' and alarmData.msgText = '%lexicon(driver:pingFail)%'
```

worked and returned a small number of records.

For example, one device had:

```text
Ping Fail = 1
```

However, this was **not** representative of the hundreds/thousands of historical communication alarm records seen on the problem devices.

The alarm details showed records with:

```text
Source State = Normal
Alarm Transition = Offnormal
Normal Time = populated
Message Text = Ping Success
```

This demonstrated that `pingSuccess` is associated with the recovery/normalisation of the communication alarm record.

### Important interpretation

Do not interpret:

```text
%lexicon(driver:pingSuccess)%
```

as:

> "This alarm record represents a successful ping event."

In the context observed here, it represents the successful recovery/normalisation associated with the device ping alarm.

This explains why a device can have hundreds or thousands of historical `pingSuccess` alarm records while also having been repeatedly going offline.

---

# 12. Failed attempt using alarmTransition

An attempt was made to filter using:

```text
alarmTransition = Offnormal
```

For example:

```text
alarm:|bql:select source as 'BACnet Device',alarmData.sourceName as 'Display Name',count('') as 'Number of Alarms over last 30 days' where period = last30Days and source like '*Drivers/BacnetNetwork*' and alarmTransition = Offnormal
```

This returned:

```text
0 rows
```

Therefore, even though `Offnormal` is visible in the Alarm Details UI, it should not be assumed that the value can be filtered in this BQL implementation using the literal:

```text
alarmTransition = Offnormal
```

### General rule

A field being visible in the Niagara UI does not prove that its displayed text is a valid BQL comparison value.

Test the actual field and syntax before using it in generated queries.

---

# 13. The key generic source filter

The following filter was eventually proven to include the D2-183 devices as well as the devices named `BacnetDevice`:

```text
source like '*Drivers/BacnetNetwork*'
```

This is much better than:

```text
source like '*Drivers/BacnetNetwork/*/BacnetDevice*'
```

because it does not assume anything about the BACnet device slot name.

The resulting source list showed devices such as:

```text
.../Drivers/BacnetNetwork/LGF-WC2-133/BacnetDevice
.../Drivers/BacnetNetwork/LGF-WC2-133/BacnetDevice1
.../Drivers/BacnetNetwork/D2-183/D2 - Compactor
.../Drivers/BacnetNetwork/D2-183/D2 - DB D2/6
.../Drivers/BacnetNetwork/D2-183/D2 - Goods Lift G2
.../Drivers/BacnetNetwork/Red Carpark-235/Red Car Park - DB-CP2
```

This proves that the source filter is not dependent on the device name.

---

# 14. Final working query

The final working query is:

```text
alarm:|bql:select alarmData.sourceName as 'BACnet Device',count('') as 'Number of Device Communication Alarms - Last 30 Days' where period = last30Days and source like '*Drivers/BacnetNetwork*' and alarmData.msgText = '%lexicon(driver:pingSuccess)%'
```

## Field-by-field explanation

### `alarm:`

Query the Niagara alarm database.

### `|bql:`

Run BQL against that alarm database.

### `select alarmData.sourceName`

Use the alarm record's stored source name as the human-readable device name.

### `as 'BACnet Device'`

Rename the output column for the operator.

### `count('')`

Count matching alarm records.

This was experimentally confirmed to work in the IQVISION 4.14 alarm database.

### `as 'Number of Device Communication Alarms - Last 30 Days'`

Give the aggregate a useful operator-facing heading.

### `where period = last30Days`

Use Niagara's dynamic last-30-days period.

This avoids a hard-coded date.

### `source like '*Drivers/BacnetNetwork*'`

Restrict the records to sources under Niagara BACnet networks.

This deliberately does **not** assume that the device slot is called `BacnetDevice`.

### `alarmData.msgText = '%lexicon(driver:pingSuccess)%'`

Restrict the results to the BACnet device ping alarm records that have the successful/recovered ping message.

This was the filter that produced the useful site-wide device communication alarm table.

---

# 15. Example result

The query returned 66 rows on the test station.

Examples included:

| BACnet Device | Number of Device Communication Alarms - Last 30 Days |
|---|---:|
| LGF-WC2-133 BacnetDevice | 1952 |
| LGF-WC2-133 BacnetDevice1 | 1676 |
| LGF-WC2-133 BacnetDevice2 | 1713 |
| LGF-WC2-133 BacnetDevice3 | 836 |
| D2-183 D2 - Basement Extract Plant (XB2) | 1747 |
| D2-183 D2 - Basement Extract Plant (XB2)1 | 14 |
| D2-183 D2 - Compactor | 158 |
| D2-183 D2 - D2/3 DB | 12 |
| D2-183 D2 - Workshop | 3 |
| D2-183 D2 - DB D2/6 | 11 |
| D2-183 D2 - DB D2G/5 | 10 |
| D2-183 D2 - DB/D2/G/9 Supply | 37 |
| D2-183 D2 - Workshop H&V | 1 |
| D2-183 D2 - Basement Drainage Sump Pump | 3 |
| D2-183 Spare | 28 |
| D2-183 D2 - Goods Lift G1 | 4 |
| D2-183 D2 - Goods Lift G2 | 28 |
| D2-183 D2 - Incoming Supply | 1 |

These values demonstrate that the query successfully identifies devices with custom names as well as devices named `BacnetDevice`.

---

# 16. Sorting

Several attempts were made to make BQL sort the aggregate by count.

These did not work correctly:

```text
order by count('') DESC
```

and:

```text
order by 2 DESC
```

Both resulted in the aggregate column being displayed as `NULL`.

Using an alias:

```text
count('') as alarmCount
...
order by alarmCount DESC
```

returned:

```text
Invalid Ord Syntax
```

### Final solution

Do **not** add an `ORDER BY` clause.

Use the known-good query and sort the resulting **Workbench Collection Table** by clicking the column heading.

This provides ascending/descending sorting without altering the BQL query.

---

# 17. Failed approaches and lessons for BQL query-generation systems

## Failed: `sourceName`

```text
select timestamp,sourceName,...
```

Result: `sourceName` was `NULL`.

### Lesson

For these alarm records, use:

```text
alarmData.sourceName
```

for the stored human-readable source name.

---

## Failed: `bqltime.last30days`

```text
where timestamp in bqltime.last30days
```

Did not work in this alarm BQL environment.

### Lesson

Do not assume generic `bqltime` expressions work against the alarm database.

---

## Worked: fixed AbsTime

```text
where timestamp >= AbsTime '2026-07-26T00:00:00.000+01:00'
```

### Lesson

Absolute timestamps work and are useful for controlled testing, but they are not ideal for reusable queries.

---

## Worked: dynamic period

```text
where period = last30Days
```

### Lesson

For this IQVISION 4.14 alarm database, this is the preferred dynamic 30-day filter.

---

## Failed: ORD period parameter

```text
alarm:?period=last30Days|bql:...
```

Result:

```text
Target Not Found
```

### Lesson

Do not move the `period` parameter into the alarm ORD query string.

Use:

```text
where period = last30Days
```

inside the BQL query.

---

## Worked: `count('')`

```text
count('')
```

### Lesson

This is the confirmed working aggregate syntax for the alarm BQL environment investigated.

Do not replace it with `count(uuid)` without testing.

---

## Failed: `source like '*.../BacnetDevice*'`

This worked for some devices but excluded devices with custom slot names.

### Lesson

Never infer Niagara component type from the component's slot name.

---

## Worked: generic BACnet source filter

```text
source like '*Drivers/BacnetNetwork*'
```

### Lesson

For site-wide BACnet communication alarm analysis, this is a useful generic source-path filter.

---

## Failed: `source.name`

```text
select source,source.name,...
```

`source.name` returned `NULL`.

### Lesson

The alarm `source` should not be assumed to be automatically dereferenced into a live Niagara component object.

---

## Worked: actual BACnet component query

```text
station:|slot:/Drivers|bql:select name,displayName,deviceId from bacnet:BacnetDevice
```

### Lesson

When the goal is to query actual BACnet device components, use the Niagara component type:

```text
bacnet:BacnetDevice
```

rather than relying on naming conventions.

---

## Failed: `ord` on `bacnet:BacnetDevice`

```text
select ord,name,displayName from bacnet:BacnetDevice
```

`ord` returned `NULL`.

### Lesson

Do not assume `ord` is available as a populated BQL property on this component type/environment.

---

## Worked: `alarmData.msgText`

This exposed:

```text
%lexicon(driver:pingSuccess)%
```

and:

```text
%lexicon(driver:pingFail)%
```

### Lesson

The alarm message is useful for distinguishing driver ping alarm records.

---

## Failed as the primary outage count: `pingFail`

Filtering:

```text
alarmData.msgText = '%lexicon(driver:pingFail)%'
```

returned very few records compared with the large number of historical communication alarm records.

### Lesson

Do not assume `pingFail` represents the historical count of outages.

In the investigated alarm records, the recovered alarm record carries:

```text
%lexicon(driver:pingSuccess)%
```

---

## Failed: `alarmTransition = Offnormal`

This returned zero rows.

### Lesson

The UI's displayed `Offnormal` value cannot automatically be assumed to be directly comparable in BQL using:

```text
alarmTransition = Offnormal
```

---

## Failed: aggregate `ORDER BY`

These approaches either produced `NULL` aggregate values or invalid syntax:

```text
order by count('') DESC
```

```text
order by 2 DESC
```

```text
order by alarmCount DESC
```

### Lesson

For this alarm BQL implementation, use the Workbench Collection Table's column sorting instead of adding `ORDER BY` to the aggregate query.

---

# 18. Important assumptions and confidence levels

## High confidence

The following are directly demonstrated by testing in IQVISION 4.14.0.162:

- `alarm:` can be queried using BQL.
- `alarmData.sourceName` contains the useful stored source name.
- `count('')` works for alarm aggregation.
- `period = last30Days` works in the alarm BQL query.
- `source` contains the Niagara source ORD.
- `source like '*Drivers/BacnetNetwork*'` identifies BACnet-network alarm sources.
- `alarmData.msgText` contains the driver ping lexicon text.
- `%lexicon(driver:pingSuccess)%` is associated with the recovered/normalised communication alarm records observed.
- BACnet devices can have arbitrary/custom Niagara slot names.
- `source.name` does not provide the live component name in this alarm query.
- BQL `ORDER BY` is not useful for this aggregate in the tested environment.

## Medium confidence

The query's count is best described as:

> Number of recovered device ping/communication alarm records in the last 30 days.

It is strongly indicative of communication interruptions because of the observed alarm behaviour, but the exact semantic meaning depends on the device's Niagara alarm configuration.

Do not automatically translate the number into an exact count of physical/network outages without inspecting the alarm configuration if absolute certainty is required.

## Not established

The investigation did **not** prove:

- That every source matching `*Drivers/BacnetNetwork*` is necessarily a `bacnet:BacnetDevice`.
- That every possible BACnet device alarm configuration uses `driver:pingSuccess`.
- That every device has the same ping/alarm configuration.
- That a count always equals the number of distinct offline incidents if alarms can be manually generated, duplicated, or otherwise configured differently.
- That the live BACnet device `displayName` can be joined directly to the alarm database source using standard BQL.

These should not be assumed by BQL query-generation systems.

---

# 19. Recommended BQL query-generation systems generation pattern

When a user asks:

> "How many times have the BACnet devices gone offline / had communication alarms in the last 30 days?"

For a Niagara 4 / IQVISION environment matching this investigation, start from:

```text
alarm:|bql:select alarmData.sourceName as 'BACnet Device',count('') as 'Number of Device Communication Alarms - Last 30 Days' where period = last30Days and source like '*Drivers/BacnetNetwork*' and alarmData.msgText = '%lexicon(driver:pingSuccess)%'
```

Do not initially:

- Hard-code a BACnet network name.
- Hard-code a device name.
- Assume the device slot is called `BacnetDevice`.
- Use `count(uuid)`.
- Use `timestamp in bqltime.last30days`.
- Use `alarm:?period=last30Days`.
- Filter on `source.name`.
- Filter on `alarmTransition = Offnormal`.
- Add `ORDER BY count('')`.
- Assume `pingFail` is the historical outage count.

Instead:

1. Identify the BQL target (`alarm:`).
2. Use `alarmData.sourceName` for the display name.
3. Use `count('')` for the aggregate.
4. Use `period = last30Days` for the dynamic time range.
5. Use `source like '*Drivers/BacnetNetwork*'` rather than a device naming convention.
6. Use the driver ping message to identify device communication alarm records.
7. Leave ordering to the Workbench Collection Table.
8. Clearly qualify what the count means if the user asks for "number of outages."

---

# 20. Useful query variants

## All BACnet communication alarm records

```text
alarm:|bql:select alarmData.sourceName as 'BACnet Device',count('') as 'Number of Device Communication Alarms - Last 30 Days' where period = last30Days and source like '*Drivers/BacnetNetwork*' and alarmData.msgText = '%lexicon(driver:pingSuccess)%'
```

## Show the actual alarm source ORD as well

```text
alarm:|bql:select source as 'BACnet Device Source',alarmData.sourceName as 'Display Name',count('') as 'Number of Device Communication Alarms - Last 30 Days' where period = last30Days and source like '*Drivers/BacnetNetwork*' and alarmData.msgText = '%lexicon(driver:pingSuccess)%'
```

## Show individual matching records rather than counts

```text
alarm:|bql:select timestamp,source,alarmData.sourceName,alarmData.msgText,sourceState,normalTime where period = last30Days and source like '*Drivers/BacnetNetwork*' and alarmData.msgText = '%lexicon(driver:pingSuccess)%'
```

This is useful when investigating a particular device and wanting to see the actual timestamps.

## Find Ping Fail records

```text
alarm:|bql:select timestamp,source,alarmData.sourceName,alarmData.msgText where period = last30Days and source like '*Drivers/BacnetNetwork*' and alarmData.msgText = '%lexicon(driver:pingFail)%'
```

This should be treated as a query for explicit `pingFail` message records, **not automatically as the total number of communication failures**.

---

# 21. Future improvements

Potential future BQL investigations could attempt to produce:

- BACnet device communication alarms for 24 hours.
- BACnet device communication alarms for 7 days.
- BACnet device communication alarms for 30 days.
- BACnet device communication alarms for 90 days.
- First communication failure in the period.
- Last communication failure in the period.
- First recovery in the period.
- Longest communication outage.
- Average outage duration.
- Currently failed BACnet devices.
- BACnet network-specific communication reports.
- BACnet device ID / network number alongside alarm counts.
- Correlation between live `bacnet:BacnetDevice` properties and historical alarm records.

These should be developed incrementally and tested against the actual Niagara version rather than assuming standard SQL behaviour.

---

# 22. Core lessons for BQL query-generation systems

The most important lessons from this investigation are:

1. **Niagara BQL is not SQL.** Do not assume SQL syntax or semantics.
2. **Test every proposed BQL construct against the actual Niagara version.**
3. **`alarmData.sourceName` is useful for alarm display names.**
4. **`source` is useful as the stored Niagara source ORD.**
5. **A component's slot name does not prove its Niagara component type.**
6. **`bacnet:BacnetDevice` should be used when querying actual BACnet device components.**
7. **`count('')` is the confirmed working aggregate syntax for this alarm BQL environment.**
8. **`period = last30Days` works where `bqltime.last30days` did not.**
9. **Do not assume a visible UI field/value can be used directly in a BQL comparison.**
10. **Driver ping alarm messages are lexicon strings and can be used as filters.**
11. **`pingSuccess` in a historical alarm record can represent recovery/normalisation, not an independent successful-ping event.**
12. **The Workbench Collection Table can perform sorting, so BQL `ORDER BY` is unnecessary for this use case.**
13. **Prefer a generic source-path filter over site-specific naming conventions when the source hierarchy is known.**
14. **When uncertain, establish what the alarm record actually contains before constructing a more complex query.**
15. **Keep a known-good query as the baseline and modify one variable at a time.**

---

# 23. Final known-good query

```text
alarm:|bql:select alarmData.sourceName as 'BACnet Device',count('') as 'Number of Device Communication Alarms - Last 30 Days' where period = last30Days and source like '*Drivers/BacnetNetwork*' and alarmData.msgText = '%lexicon(driver:pingSuccess)%'
```

**Status: VERIFIED WORKING on IQVISION 4.14.0.162 during this investigation.**
