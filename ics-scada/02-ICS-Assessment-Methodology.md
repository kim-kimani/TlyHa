# ICS Assessment Methodology

This is how a real industrial control assessment is actually run, phase by phase, with the goal of each phase and the safety constraint that shapes it. Unlike web or even IT network testing, the method here is defined as much by what you must not do as by what you do, because the target controls a physical process. For a newcomer, the most important lesson is that ICS assessment is mostly careful observation, not aggressive exploitation.

## The mental model

An ICS assessment answers one overarching question: can an attacker get from somewhere they can reach (the internet, the corporate IT network, a vendor connection, a USB port) to a place where they can influence the physical process, and would anyone notice? Because the control protocols have no security, the hard part for an attacker, and therefore the focus of the assessment, is crossing the boundaries into the control network. So you spend your effort on architecture, segmentation, and monitoring, and you touch the fragile control devices as little as possible.

## The phases, each with its goal and its safety rule

### Phase 0: scoping and safety agreement (goal: define what is testable and how)

Before anything technical: agree with the operator what is in scope, what is off-limits (almost always: no active testing of live production controllers and never the safety systems), the test windows, who is present, and the rollback/abort plan. In ICS this is not bureaucracy; it is how you avoid causing an outage or a safety event. The goal is a plan that can produce findings without risking the process.

### Phase 1: architecture and documentation review (goal: understand the design on paper)

What to aim for: a map of the network zones (the Purdue levels), every connection between IT and OT, remote-access paths, firewall rules, and the device inventory, from diagrams and interviews rather than scanning. Most high-value findings (a flat network, a dual-homed historian, an always-on vendor VPN, a firewall rule allowing IT to reach PLCs) are visible here without sending a single packet to a PLC. This is the safest and often most productive phase.

### Phase 2: passive traffic analysis (goal: see the real network without disturbing it)

What to aim for: identify devices, protocols, and communication pairs by listening, not probing. A network tap or a SPAN/mirror port feeds a passive analyzer; you learn which HMIs talk to which PLCs, what protocols are in use, and whether anything is exposed or misconfigured, all with zero risk to the devices.

```
# passive capture on a mirror/tap, analyzed offline:
#   Wireshark with ICS dissectors, or an OT-aware tool, to inventory devices and
#   protocols (Modbus/S7/DNP3/ENIP) and map master<->outstation relationships.
# passive asset-discovery tools (e.g. GRASSMARLIN-style) build a map from traffic alone.
```

### Phase 3: careful, authorized active discovery (goal: confirm exposure, gently)

What to aim for: verify reachability and identify devices from the IT/boundary side, slowly and only where agreed, because scanning can crash old PLCs.

```
# slow, limited host discovery (never a full aggressive scan of a control network):
nmap -sn <boundary-segment>                 # ping sweep only, rate-limited
# targeted ICS port/identity checks on agreed hosts, carefully:
nmap -p 502,102,20000,44818,47808 --scan-delay 1s <host>
# device-identity reads (read-only) to fingerprint vendor/model/firmware.
# STOP if any device shows instability; active scanning is the riskiest routine step.
```

### Phase 4: boundary and remote-access testing (goal: the real attack path)

What to aim for: test whether the IT/OT boundary actually holds. This is where the genuine risk lives, so it is where testing focus belongs: can you reach OT from IT, through the firewall rules, the jump host, the historian, the engineering workstation, or the vendor remote-access solution? These are usually tested with normal IT techniques (which are safer than touching PLCs) against the boundary systems, not the controllers.

### Phase 5: controlled device/protocol testing (goal: demonstrate impact, only on safe targets)

What to aim for: show that an attacker on the control network could read and command devices, demonstrated on offline/spare equipment, a test bench, or a simulator (see `00-ICS-Primer-and-Safe-Lab.md`), not on the live process. On live systems this is usually limited to passive proof and read-only interactions explicitly authorized. The deeper techniques (register writes, PLC logic download, HMI compromise) are in `PLC-and-HMI-Attacks.md` and are practiced in the lab.

### Phase 6: reporting (goal: actionable, risk-ranked, defender-focused)

What to aim for: findings framed by the process risk and the architectural fix (segment, monitor, lock down remote access), because you usually cannot "patch" an unauthenticated protocol. Prioritize the boundary and monitoring gaps that would let an attacker reach and influence the process undetected.

## Beyond the basics

- **The finding is almost always the boundary, not the PLC**: since the protocols cannot authenticate, "the PLC accepts unauthenticated writes" is a given, not a discovery. The valuable findings are the paths that let an attacker reach the PLC (IT/OT bridges, remote access, flat networks, exposed devices) and the absence of detection. Spend your effort there.
- **Passive-first is a discipline, not a limitation**: in OT, how much you can learn without sending packets to control devices is a core skill. Mature OT assessment leans heavily on taps, mirror ports, and traffic analysis, reserving active techniques for the boundary and for offline equipment.
- **Safety instrumented systems are off-limits**: the independent controllers that trip the process to a safe state are the hard red line. Attacks on them (as in the Triton/Trisis incident) risk catastrophe; in assessments they are reviewed architecturally, not actively tested, except in dedicated offline settings.
- **Monitoring and detection are part of the assessment**: a key question is "would anyone see this?" OT-specific intrusion detection (which flags unusual write/program commands and new talkers) is both a finding (its absence) and something to validate.
- **Know the standards**: real engagements map to frameworks like IEC 62443 and the Purdue model; findings land better when tied to a recognized reference the operator already follows.
- **Shodan/Censys show the external exposure (passively)**: internet-exposed ICS is a real and serious finding category, discoverable through search-engine metadata without touching the devices. Looking is passive; connecting is not, and you do not connect to systems you are not authorized to test.

## What to watch for (safety framing, the whole point)

- Active scanning and especially fuzzing can crash fragile controllers and disrupt a physical process; default to passive, go active only where authorized, slowly, with operators present and an abort plan.
- Never actively test safety systems or live production controllers without an explicit, narrow, supervised authorization; prefer offline/duplicate equipment and simulators.
- Reachability of an unauthenticated protocol is expected; the actionable issue is the missing isolation and monitoring, so report architecture, not protocol "bugs."
- The objective is defensive: help the operator segment, monitor, and lock down. Keep that framing in every phase.

## Checklist

- Phase 0: scope, safety agreement, abort plan; safety systems out of scope for active testing.
- Phase 1: review architecture, zones, IT/OT connections, remote access, inventory, on paper.
- Phase 2: passive traffic analysis via tap/mirror; map devices and protocols with zero risk.
- Phase 3: gentle, authorized active discovery only where agreed; stop at any instability.
- Phase 4: focus real testing on the IT/OT boundary and remote-access paths.
- Phase 5: demonstrate device/protocol impact on offline equipment or simulators, not the live process.
- Phase 6: report risk-ranked, architecture-focused, defender-oriented findings.
