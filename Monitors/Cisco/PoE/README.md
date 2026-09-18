# Cisco Catalyst PoE Operational Collection

Last updated: 2026-09-18

SNMP validated against: Catalyst 3850 running IOS XE 16.12.14, with two PSE
groups and 96 PoE ports, using SNMPv3 authentication and privacy.

ThirdEye import and scheduled collection have not yet been validated.

This example collects Power over Ethernet (PoE) capacity, consumption, budget
headroom, per-port power delivery state, and fault counters. It was developed
for PoE-capable Catalyst 3850 switches. Other Cisco platforms may implement
the same MIBs, but must be validated independently.

The collection is read-only. It does not enable or disable PoE, change port
priorities, configure power policing, or install alert policies or dashboards.

## Requirements

- The switch is present in ThirdEye inventory and reachable over SNMP,
  normally UDP port 161.
- A working SNMP credential is assigned to the switch. SNMPv3 authentication
  and privacy are recommended.
- The SNMP view permits both `1.3.6.1.2.1.105` and
  `1.3.6.1.4.1.9.9.402`.
- `POWER-ETHERNET-MIB` and `CISCO-POWER-ETHERNET-EXT-MIB`, including their
  imported dependencies, are installed in ThirdEye.
- The hardware supports PoE and exposes the required tables. Non-PoE
  Catalyst models are not appropriate targets.

The vendor MIBs and credentials are not included. Obtain the MIBs from the
[Cisco MIB download site](https://cfnng.cisco.com/mibs) or Cisco's official
[MIB repository](https://github.com/cisco/cisco-mibs/tree/main/v2).

## Installation

1. Import the required MIB modules and dependencies into ThirdEye.
2. In **Inventory**, select the switch and run **Tools > SNMP System Info**.
   Confirm that the query succeeds using its assigned SNMP credential.
3. Under **Monitors > Sets**, import
   `Cisco Catalyst PoE - Operational Collection.3em`.
4. Assign **Cisco Catalyst PoE - Operational Collection** to the intended
   PoE-capable switches.
5. Wait at least one collection interval, then inspect the four monitors
   under the device's **Monitors** view.

Automatic assignment is disabled. All four monitors poll every minute and
use three-month retention, following the other operational examples.

## Included monitors

### Cisco PoE - PSE Capacity and Consumption

Collects `POWER-ETHERNET-MIB::pethMainPseTable`, indexed by PSE group:

- `pethMainPsePower`: nominal capacity in watts.
- `pethMainPseOperStatus`: on, off, or faulty.
- `pethMainPseConsumptionPower`: measured consumption in watts.
- `pethMainPseUsageThreshold`: the switch's configured usage threshold,
  expressed as a percentage. Collecting it does not create a ThirdEye alert.

### Cisco PoE - PSE Budget and Headroom

Collects `CISCO-POWER-ETHERNET-EXT-MIB::cpeExtMainPseTable`, indexed by the
same PSE group:

- `cpeExtMainPseEntPhyIndex`: corresponding ENTITY-MIB physical index.
- `cpeExtMainPseDescr`: PSE description.
- `cpeExtMainPsePwrMonitorCapable`: power-monitoring capability.
- `cpeExtMainPseUsedPower`: Cisco-reported used budget in milliwatts.
- `cpeExtMainPseRemainingPower`: remaining budget in milliwatts.

Keep budget usage separate from measured consumption. On the validation
switch, one group reported approximately 22 W of measured consumption but
51.6 W of used budget. These values are not interchangeable. For budget
utilization, use the Cisco used/remaining values together; do not substitute
measured consumption for used budget when assessing headroom.

### Cisco PoE - Port State and Faults

Collects `POWER-ETHERNET-MIB::pethPsePortTable`, indexed by
`pethPsePortGroupIndex.pethPsePortIndex`:

- `pethPsePortAdminEnable`
- `pethPsePortDetectionStatus`
- `pethPsePortPowerPriority`
- `pethPsePortPowerClassifications`
- `pethPsePortMPSAbsentCounter`
- `pethPsePortInvalidSignatureCounter`
- `pethPsePortPowerDeniedCounter`
- `pethPsePortOverLoadCounter`
- `pethPsePortShortCounter`

An enabled, unused port normally reports `searching`, not `deliveringPower`.
Do not treat every non-delivering port as a fault. Power classification is
meaningful only while the port is delivering power. The MIB enumeration
values are one-based: `class0` is value 1, through `class4` as value 5.

Fault and MPS-absent values are cumulative Counter32 objects, not present-state
flags. Look for increases over a baseline, accounting for counter wraps and
resets; a historical nonzero value does not establish an ongoing fault.
Confirm whether the ThirdEye view displays the counter or a derived rate
before selecting an alert condition. MPS absence can reflect device removal
and is not necessarily a hardware failure.

### Cisco PoE - Port Power

Collects `CISCO-POWER-ETHERNET-EXT-MIB::cpeExtPsePortTable`, using the same
compound group/port index:

- `cpeExtPsePortDeviceDetected`
- `cpeExtPsePortPwrAllocated`
- `cpeExtPsePortPwrAvailable`
- `cpeExtPsePortPwrConsumption`
- `cpeExtPsePortMaxPwrDrawn`
- `cpeExtPsePortEntPhyIndex`

The four power fields are milliwatts; divide by 1,000 to display watts.
Allocated/available power is not necessarily actual consumption. Maximum
draw is the peak since the powered device was powered on, not the peak for
the current chart interval. These fields are absolute readings, not counters
to differentiate into rates.

## Port identity and stack handling

Preserve the complete `group.port` index. Port 1 in group 1 and port 1 in
group 2 are different ports. PSE indices are not `IF-MIB::ifIndex` values.

The Cisco extension provides `cpeExtPsePortEntPhyIndex`, which can be matched
to `ENTITY-MIB::entPhysicalName` for a physical interface label. Validate the
mapping rather than assuming an arithmetic conversion. On the test switch,
the first port in each group mapped to physical indices 1062 and 2062.
Physical index zero means no mapping is supplied.

The bundle retains the physical index but does not collect the ENTITY-MIB
inventory or perform an automatic join to interface names. Stack membership
changes can alter indices and invalidate previously established labels.

## Suggested panels

The bundle supplies collection only. Useful customer-created panels include:

- Used and remaining PoE budget per PSE group, converted to watts.
- Actual per-port consumption, sorted highest first, alongside allocation.
- Port delivery state and changes in denied-power, overload, and short counters.

Derived percentages, unit conversions, cross-monitor joins, and fault-counter
deltas are not embedded in this example. Sum by compatible PSE group only;
do not assume these tables fully describe StackPower sharing or redundancy.

## Validation and limitations

Live read-only SNMP validation returned two PSE groups, 96 ports, positive
draw on two ports, and physical port mappings. This establishes that the
source OIDs are available on the tested release, not that the bundle has
completed ThirdEye import or end-to-end panel validation. No overload,
short-circuit, or denied-power event was deliberately induced.

After import, compare capacity, allocation, consumption, and port state with
`show power inline` on the switch. Check every stack member, confirm the port
labels, and verify unit conversions. Establish a baseline before choosing
thresholds; an empty port is not inherently an incident.

If general SNMP access works but the PoE tables are empty, check hardware
capability, the SNMP view, and support in the installed IOS XE release. If
the standard tables work but Cisco-specific metrics are absent, review the
extension MIB and platform support rather than treating missing data as zero.

For platform behavior, see Cisco's
[Catalyst 3850 PoE configuration guide](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3850/software/release/37e/consolidated_guide/b_37e_consolidated_3850_cg/b_37e_consolidated_3850_cg_chapter_01010.html).
