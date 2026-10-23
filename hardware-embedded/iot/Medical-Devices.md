# Medical Devices

Connected medical devices (implantable pacemakers and defibrillators, insulin pumps, hospital infusion pumps, patient monitors) are life-critical computers with radios and, often, decades-old software. This guide is written for understanding and defending them, because the stakes are human life. Real work here is done under strict authorization, with vendors and regulators, and through coordinated disclosure, never against devices in use on patients.

## What it is and why it matters (the aim, defensively)

The aim of research here is to find and close flaws before they harm anyone. The categories that matter:

- **Therapy manipulation**: implantable and infusion devices deliver therapy (a shock, a pacing signal, a drug dose). The nightmare is unauthorized or altered therapy. Research has demonstrated, in labs, wireless commands to deliver a shock or change insulin/infusion dosing. Understanding this drives the fix.
- **Data exposure**: devices and their telemetry carry sensitive health data. Aim: identify leakage.
- **Availability**: a crashed or drained device is a safety event. Aim: find robustness/battery-drain flaws.
- **Fleet/clinical-network exposure**: hospital devices on flat networks with known IT vulnerabilities. Aim: find the segmentation and patching gaps.

## Architecture (the sides)

- **Device**: an embedded MCU/RTOS with extreme reliability constraints and (historically) little security; implants use ultra-low-power radios.
- **Radio**: implants use MICS/MedRadio bands and legacy proprietary links, and increasingly BLE; pumps/monitors use Wi-Fi/Ethernet and sometimes BLE.
- **Programmer/clinician device**: a dedicated programmer (for implants) or a hospital system that configures/reads the device; a key trust element.
- **Cloud/clinical systems**: remote monitoring, EHR integration, and vendor cloud.

## How this is studied (defensively, on authorized test units)

### Implantable devices (pacemakers, ICDs, pumps)

Research on explanted/test units examines the wireless telemetry between the programmer/home-monitor and the device: is it authenticated and encrypted, or can commands be replayed/forged. The historical findings (e.g. the research that led to recalls and firmware fixes) were that some links lacked authentication, so a nearby radio could issue commands or alter settings. The defensive takeaway is authenticated, encrypted telemetry and proximity/pairing controls.

### Insulin and infusion pumps

Studied for remote dosing commands over their wireless links and for their network/management interfaces. Several infusion pumps had hardcoded credentials, unauthenticated network services, and update mechanisms without verification, which allowed configuration/firmware tampering on the test bench. The fixes are authentication, signed firmware, and network segmentation.

### Hospital devices (monitors, imaging, pumps)

These are often normal Windows/Linux systems on the clinical network, carrying standard IT vulnerabilities and the healthcare protocols HL7 and DICOM. Assessment (authorized, in clinical-engineering settings) looks at segmentation, patch status, default credentials, and the HL7/DICOM interfaces, which have historically lacked authentication and can leak or alter patient data/images.

## Beyond the basics (what the research teaches)

- **Legacy radios and missing authentication**: the recurring theme in implant research is wireless control links that were not designed to resist a malicious transmitter. Modern devices add authentication, encryption, and pairing; assessing whether those are present and correct is the core question.
- **Battery-drain and DoS as safety events**: forcing an implant to stay in a high-power telemetry mode to drain its battery, or crashing a pump, is a safety issue even without therapy manipulation. Robustness matters as much as confidentiality.
- **HL7/DICOM and the clinical network**: these protocols often have no authentication; on a flat network an attacker can read/modify orders, results, and images. DICOM files have even been shown to carry executable payloads. Segmentation and protocol security are the defenses.
- **Supply chain and update integrity**: unsigned firmware/updates on pumps and monitors allow persistent tampering; signed, verified updates are essential.
- **Coordinated disclosure and regulation**: this field has mature processes (FDA premarket/postmarket cybersecurity guidance, ICS-CERT/medical advisories, manufacturer disclosure programs). The impact of research here is measured in recalls and patches, not exploits.

## What to watch for (this is the part that matters most)

- Never test medical devices that are in use on or implanted in a person, and never test on a clinical network without explicit, formal authorization from the organization and following its clinical-engineering process. The risk is human harm.
- Work only on explanted, test, or vendor-provided units in a controlled lab, under authorization, with coordinated disclosure to the manufacturer and relevant authorities.
- Radio transmission near these bands is regulated and, near real devices, dangerous; understand the legal and safety constraints.
- Frame all findings around patient safety and defense: the goal is to get vulnerabilities fixed through the proper channels.
- To defend (the objective): authenticated and encrypted device telemetry, pairing/proximity controls, signed and verified firmware, no hardcoded credentials, segmented and monitored clinical networks, secured HL7/DICOM, and participation in coordinated disclosure.

## Checklist (defensive/learning)

- Understand the device classes (implants, pumps, hospital systems) and their radios/interfaces.
- For any assessment, confirm formal authorization, use only non-clinical test units, and follow coordinated disclosure.
- Examine wireless links for authentication/encryption and replay resistance (authorized test units only).
- Examine network services, HL7/DICOM, default credentials, and firmware-update integrity.
- Consider availability/battery-drain as safety findings.
- Report through manufacturer and regulatory channels; measure success by fixes, not exploits.
