# Device Power Management Policy Configuration

A production-grade Microsoft Intune configuration profile that enforces standardised power management and lockscreen security policies across a Windows 10/11 enterprise fleet. This project covers display timeout control, system sleep orchestration, physical button behaviour suppression, hard disk persistence, and authentication-on-resume enforcement — all deployed through the Intune Settings Catalog.

---

## Overview

Unmanaged power settings create a surprisingly broad attack surface in enterprise environments. Devices that never lock expose sensitive data to visual eavesdropping. Endpoints that sleep or shut down unpredictably disrupt remote management, patch compliance, and security log continuity. Left to end-user discretion, power behaviour varies wildly across an estate — making it nearly impossible to guarantee a consistent security posture.

This project delivers a single, unified Intune configuration profile that centralises power management policy across every enrolled Windows endpoint. Rather than scattering settings across multiple GPOs or relying on local power plan manipulation, the entire power management stack is governed through one Settings Catalog profile — providing a single pane of glass for deployment, monitoring, and compliance reporting.

The policy targets five core areas: display timeout, system sleep, physical button behaviour, hard disk power state, and authentication on wake. Each setting is engineered to balance security enforcement with operational usability, ensuring devices remain protected without degrading the end-user experience.

---

## Objectives

- **Standardise power behaviour** across all managed Windows endpoints, eliminating configuration drift caused by local user changes or imaging inconsistencies.
- **Reduce the idle attack surface** by enforcing automatic display timeout and system sleep, ensuring unattended devices are never left in an exposed state.
- **Prevent accidental shutdowns and sleep events** by suppressing physical button and lid-close actions, protecting users from unintended interruptions during critical work.
- **Maintain disk and log availability** by disabling automatic hard disk spin-down, ensuring security telemetry, compliance agents, and background services operate without interruption.
- **Enforce authentication on resume** so that every transition from sleep, hibernate, or screen timeout requires credential verification — closing the post-idle access gap.
- **Simplify fleet governance** through a single Intune configuration profile that can be targeted, versioned, and audited from one location.

---

## Technologies & Tools Used

| Component | Detail |
|---|---|
| **Management Platform** | Microsoft Intune (Endpoint Manager) |
| **Profile Type** | Settings Catalog |
| **Target OS** | Windows 10 & later |
| **Policy Source** | Administrative Templates (ADMX-backed) + Power CSP |
| **Assignment Method** | Entra ID security groups / All Devices |
| **Monitoring** | Intune device configuration reporting |

---

## Environment / Prerequisites

Before deploying this configuration, the following must be in place:

- **Microsoft Intune** — active tenant with device configuration permissions (Intune Administrator or equivalent RBAC role).
- **Entra ID (Azure AD)** — devices must be either Entra ID joined or Hybrid Entra ID joined to receive Intune policy.
- **Windows 10/11 enrolment** — target endpoints must be enrolled in Intune MDM (Autopilot, bulk enrolment, or manual join).
- **Licensing** — Microsoft Intune Plan 1 (included in Microsoft 365 E3/E5, Business Premium, or standalone).
- **Security group** — a dedicated Entra ID security group for policy targeting (e.g., `SG-Intune-PowerManagement-Policy`), or use "All Devices" for estate-wide deployment.

---

## Architecture / Policy Design

The policy is built as a **single Settings Catalog profile** in Intune, consolidating five distinct power management domains into one deployable unit. This architectural decision is deliberate — rather than creating separate profiles for display, sleep, buttons, disk, and authentication (which increases management overhead and introduces assignment conflicts), a single profile ensures atomic deployment and simplifies troubleshooting.

### Policy Structure

```
Device Power Management & LockScreen Policy
├── Video and Display Settings
│   ├── Turn off display (on battery)      → 300 seconds
│   └── Turn off display (plugged in)      → 300 seconds
├── Sleep Settings
│   ├── System sleep timeout (on battery)  → 600 seconds
│   ├── System sleep timeout (plugged in)  → 600 seconds
│   ├── Require password on wake (battery) → Enabled
│   └── Require password on wake (AC)      → Enabled
├── Power (Button & Lid Behaviour)
│   ├── Power button action (AC / battery) → Do Nothing
│   ├── Sleep button action (AC / battery) → Do Nothing
│   └── Lid close action (AC / battery)    → Do Nothing
├── Hard Disk Settings
│   ├── Turn off hard disk (on battery)    → Disabled
│   └── Turn off hard disk (plugged in)    → Disabled
└── Sleep Settings (Authentication)
    └── Prompt for password on resume       → Enabled
```

### Design Rationale

The **5-minute display timeout** was selected as the enterprise sweet spot — short enough to mitigate visual exposure on unattended devices, long enough to avoid frustrating users during natural pauses (reading documents, attending to a colleague). The **10-minute sleep timer** layers on top, giving the system an additional 5-minute window after display-off before entering a low-power state. This staggered approach means users who return within 5 minutes simply move the mouse to reactivate the display; users away for longer face a full credential prompt on wake — a proportional security escalation.

Setting all **physical buttons and lid actions to "Do Nothing"** is a deliberate enterprise hardening decision. In shared workspaces, hot-desking environments, and docking station setups, accidental lid closures or button presses can trigger unexpected sleep or shutdown — terminating VPN tunnels, killing active file transfers, or interrupting patch installations. By neutralising these triggers, the device's power state is governed exclusively by the idle timers defined in policy, not by physical interaction.

**Hard disk spin-down is disabled** to maintain continuous availability for background services that depend on persistent disk access — including endpoint protection agents, compliance evaluation, Windows Update servicing, and security event logging. On devices with traditional HDDs, unexpected spin-down introduces latency when the disk must restart, which can cause agent timeouts and missed telemetry.

---

## Step-by-Step Implementation

### 1. Create the Configuration Profile

Navigate to the **Microsoft Intune admin centre** and create a new Settings Catalog profile. The Settings Catalog is used instead of legacy Templates because it exposes the full breadth of ADMX-backed and CSP-based settings, supports granular conflict detection, and aligns with Microsoft's forward-looking policy management strategy.

- Open **Microsoft Intune admin centre** → `Devices` → `Configuration` → `Create` → `+ New Policy`
- **Platform:** Windows 10 and later
- **Profile type:** Settings Catalog
- Click **Create**

### 2. Configure Basics

Define the profile identity. A clear, descriptive name is critical for operational clarity — when an estate runs dozens of configuration profiles, ambiguous names create confusion during troubleshooting and audit.

- **Name:** `Device Power Management & LockScreen Policy`
- **Description:** `Screen, sleep, hibernate time-out settings + Lid, Power & Sleep button control settings`

### 3. Add Display Timeout Settings

These settings control how quickly the display powers off after user inactivity. A 5-minute timeout balances security (limiting the window of visual exposure) with usability (avoiding premature screen blanking during legitimate work pauses).

- Click **+ Add settings**
- Browse to: `Administrative Templates` → `System` → `Power Management` → `Video and Display Settings`
- Enable the following:

| Setting | Value |
|---|---|
| Turn off the display (on battery) | **Enabled** |
| On battery power, turn display off after (seconds) | **300** |
| Turn off the display (plugged in) | **Enabled** |
| When plugged in, turn display off after (seconds) | **300** |

### 4. Add Sleep Timeout Settings

Sleep settings define when the system transitions to a low-power state. The 600-second (10-minute) timer provides a deliberate buffer after the display timeout — if the screen has been off for 5 minutes and the user still hasn't returned, the system enters sleep. This staggered design reduces unnecessary sleep/wake cycles for users who step away briefly, while ensuring genuinely idle devices are secured.

- Click **+ Add settings**
- Browse to: `Administrative Templates` → `System` → `Power Management` → `Sleep Settings`
- Enable the following:

| Setting | Value |
|---|---|
| Specify the system sleep timeout (on battery) | **Enabled** |
| System Sleep Timeout (seconds) | **600** |
| Specify the system sleep timeout (plugged in) | **Enabled** |
| System Sleep Timeout (seconds) | **600** |

### 5. Configure Lid, Power, and Sleep Button Behaviour

Suppressing physical button and lid actions prevents users or environmental factors from overriding the policy-controlled power state. This is particularly important in hot-desking environments and docking station configurations, where lid closure is a normal part of the workflow — not an intent to sleep.

- Click **+ Add settings**
- Browse to: `Power` (under the Power category — **not** under Administrative Templates)
- Set all actions to suppress default behaviour:

| Setting | Value |
|---|---|
| Select Power Button Action (Plugged In) | **Take no action** |
| Select Power Button Action (On Battery) | **Take no action** |
| Select Sleep Button Action (Plugged In) | **Take no action** |
| Select Sleep Button Action (On Battery) | **Take no action** |
| Select Lid Close Action (Plugged In) | **Take no action** |
| Select Lid Close Action (On Battery) | **Take no action** |
| Energy Saver Battery Threshold (On Battery) | **0** |

### 6. Add Hard Disk Power Settings

Disabling automatic hard disk shutdown ensures background agents and logging services maintain uninterrupted disk access. This prevents compliance evaluation failures, security log gaps, and the latency penalties associated with disk spin-up on traditional HDDs.

- Click **+ Add settings**
- Browse to: `Administrative Templates` → `System` → `Power Management` → `Hard Disk Settings`
- Configure:

| Setting | Value |
|---|---|
| Turn Off the hard disk (on battery) | **Disabled** |
| Turn Off the hard disk (plugged in) | **Disabled** |

### 7. Enable Authentication on Resume

This is the security linchpin of the entire policy. Without password-on-wake enforcement, the display and sleep timers only reduce power consumption — they don't protect data. Enabling authentication on resume ensures that every idle-to-active transition requires credential verification, closing the post-idle access gap.

- Remain in the **Sleep Settings** subcategory (or re-add it)
- Enable the following:

| Setting | Value |
|---|---|
| Prompt for password on resume from hibernate/suspend (User) | **Enabled** |
| Require a password when a computer wakes (on battery) | **Enabled** |
| Require a password when a computer wakes (plugged in) | **Enabled** |

### 8. Assign and Deploy

- Navigate to the **Assignments** tab
- Target the profile to a dedicated Entra ID security group (e.g., `SG-Intune-PowerManagement-Policy`) or assign to **All Devices** for estate-wide enforcement
- Click **Review + Create** and confirm deployment

---

## Configuration Details

### Screen Timeout Policies

| Parameter | Battery | Plugged In |
|---|---|---|
| Display off after | 300 seconds (5 min) | 300 seconds (5 min) |
| CSP Path | `Administrative Templates\System\Power Management\Video and Display Settings` | Same |

**What it does:** Automatically powers off the display after 5 minutes of user inactivity, regardless of power source.

**Why it matters:** An active display on an unattended device is an open invitation for visual data exposure — whether in an open-plan office, a client site, or a shared workspace. The 5-minute threshold was selected because it sits at the intersection of security and usability: short enough to limit exposure windows, long enough to accommodate natural work pauses without triggering frustration-driven policy complaints.

**Security implication:** Display timeout alone does not lock the session. It must be paired with authentication-on-resume to provide genuine protection. Without the credential requirement, waking the display simply restores the previous session in full — making display timeout a power-saving measure, not a security control.

### Sleep Policies

| Parameter | Battery | Plugged In |
|---|---|---|
| System sleep after | 600 seconds (10 min) | 600 seconds (10 min) |
| CSP Path | `Administrative Templates\System\Power Management\Sleep Settings` | Same |

**What it does:** Transitions the device into system sleep (S3 standby) after 10 minutes of inactivity.

**Why it matters:** Sleep serves a dual purpose — it reduces energy consumption and, when paired with authentication-on-resume, acts as a security enforcement point. The 10-minute timer is deliberately staggered 5 minutes after the display timeout. This means the device follows a predictable idle progression: display off at 5 minutes, system sleep at 10 minutes. Users returning within the first 5 minutes simply reactivate the display; those returning after 10 minutes must authenticate. This proportional escalation minimises disruption for brief absences while securing the device during extended ones.

**Performance consideration:** The 600-second value is not a universal standard — it serves as a strong baseline for most enterprise environments. Organisations with heightened security requirements (financial services, healthcare, government) may reduce this to 300 seconds. Environments where users frequently reference physical documents alongside their screen may benefit from extending to 900 seconds. The values should be tuned to the organisation's risk profile.

### Lid / Power Button Behaviour

| Action | Plugged In | On Battery |
|---|---|---|
| Power button press | Do Nothing | Do Nothing |
| Sleep button press | Do Nothing | Do Nothing |
| Lid close | Do Nothing | Do Nothing |

**What it does:** Suppresses all default system responses to physical power interactions. Closing the lid, pressing the power button, or pressing the sleep button triggers no power state change.

**Why it matters:** In enterprise environments — particularly those using docking stations, multi-monitor setups, or hot-desking configurations — the default Windows lid-close behaviour (sleep or hibernate) creates operational disruption. Users who close their laptop lid while connected to an external display expect the system to continue running. Default button mappings can also cause accidental shutdowns during critical operations such as software deployments, large file transfers, or compliance scans.

**Security implication:** Neutralising physical buttons means the device's power state is governed entirely by the idle timers defined in this policy. This creates a predictable, auditable power lifecycle: the device will always follow the display-off → sleep → authenticate sequence, regardless of physical interaction. No user action can bypass the configured idle security chain.

### Hard Disk Management

| Parameter | Battery | Plugged In |
|---|---|---|
| Auto hard disk shutdown | Disabled | Disabled |
| CSP Path | `Administrative Templates\System\Power Management\Hard Disk Settings` | Same |

**What it does:** Prevents Windows from automatically spinning down the hard disk after a period of inactivity.

**Why it matters:** Enterprise endpoints run a continuous background workload that most users never see — endpoint protection real-time scanning, Intune compliance evaluation, Windows Update content delivery, security event log writing, and BitLocker encryption operations. All of these require persistent disk access. When the OS spins down the disk to save power and a background agent needs to write, the resulting spin-up delay can cause agent timeouts, missed telemetry windows, and compliance evaluation failures that report a healthy device as non-compliant.

**Performance consideration:** On devices equipped with SSDs, the performance impact of this setting is negligible since SSDs have no mechanical spin-up penalty. On devices with traditional HDDs, disabling spin-down ensures consistent I/O response times at the cost of marginally higher power consumption — a worthwhile trade-off for operational reliability.

### Authentication on Resume

| Parameter | Value |
|---|---|
| Prompt for password on resume from hibernate/suspend (User) | Enabled |
| Require a password when a computer wakes (on battery) | Enabled |
| Require a password when a computer wakes (plugged in) | Enabled |
| CSP Path | `Administrative Templates\System\Power Management\Sleep Settings` |

**What it does:** Forces a credential prompt (password, PIN, Windows Hello) every time the device transitions from sleep, hibernate, or screen timeout back to an active state.

**Why it matters:** This is the single most critical setting in the entire policy. Every other configuration — display timeout, sleep timer, button suppression — creates the conditions for a device to enter an idle state. Authentication on resume is what makes that idle state a security boundary. Without it, an attacker (or unauthorised colleague) who encounters a sleeping device can simply wake it and gain full session access. With it, the idle state becomes equivalent to a locked workstation.

**Compliance alignment:** Password-on-resume enforcement is a baseline requirement in virtually every enterprise security framework, including CIS Benchmarks for Windows, NIST 800-171, ISO 27001 Annex A, and Cyber Essentials Plus. Failure to enforce this setting is a common audit finding and a frequent vector in insider threat scenarios.

---

## Testing & Validation

### Pre-Deployment Validation

1. **Profile conflict check** — Before deployment, verify that no existing Intune profiles or legacy GPOs configure overlapping power management settings. Use Intune's `Settings Catalog` conflict detection view and run `gpresult /h` on a test device to identify any competing policies.

2. **Pilot group deployment** — Assign the profile to a small Entra ID security group containing representative hardware (laptops, desktops, docked devices, undocked devices) before rolling out estate-wide.

### Post-Deployment Verification

3. **Intune reporting** — Monitor `Devices` → `Configuration` → select the profile → `Device status`. Confirm all targeted devices report **Succeeded**. Investigate any **Error** or **Conflict** states before expanding deployment.

4. **Client-side validation** — On a test device, verify the applied policy:
   - Run `powercfg /query` in an elevated command prompt to confirm active power scheme values match the configured thresholds.
   - Let the device sit idle and verify the display powers off at 5 minutes and the system sleeps at 10 minutes.
   - Wake the device and confirm a credential prompt appears.
   - Close the lid and verify the system does not enter sleep.
   - Press the power button and verify no shutdown or sleep occurs.

5. **Event log verification** — Check `Event Viewer` → `Applications and Services Logs` → `Microsoft` → `Windows` → `Power-Troubleshooter` to confirm sleep/wake events align with the configured timers.

6. **Registry validation** — Confirm Intune has written the expected values:
   ```
   HKLM\SOFTWARE\Policies\Microsoft\Power\PowerSettings
   ```

---

## Results / Outcomes

- **Consistent power behaviour** across all enrolled Windows endpoints — no device-level variation or user-modified power plans.
- **Automated idle security** — every unattended device follows a predictable display-off → sleep → credential-prompt sequence within 10 minutes of inactivity.
- **Zero accidental power events** — lid closures and button presses no longer disrupt active work, VPN tunnels, or background operations.
- **Uninterrupted background services** — endpoint protection, compliance agents, and security logging maintain continuous disk access.
- **Audit-ready configuration** — the policy satisfies authentication-on-resume requirements across CIS, NIST, ISO 27001, and Cyber Essentials frameworks.
- **Single-profile management** — one Intune configuration profile governs the entire power management stack, simplifying versioning, targeting, and troubleshooting.

---

## Challenges & Solutions

### Challenge: Overlapping Legacy GPOs

**Problem:** Existing Group Policy Objects from a previous on-premises Active Directory environment were applying conflicting power settings to Hybrid Entra ID joined devices, causing Intune policy application failures.

**Solution:** Conducted a `gpresult /h` audit across representative devices to identify competing policies. Worked with the infrastructure team to either remove the conflicting GPO settings or apply WMI filters to exclude Intune-managed devices from the legacy GPO scope. Intune's built-in conflict detection in the Settings Catalog was then used to confirm clean application.

### Challenge: User Pushback on Sleep Timers

**Problem:** A subset of users in data analysis and software development roles reported that the 10-minute sleep timer interrupted long-running processes that required no direct user interaction (compiling, data processing, model training).

**Solution:** Created a secondary power management profile with an extended sleep timeout (30 minutes) and assigned it to a dedicated Entra ID security group for power users. The authentication-on-resume and button suppression settings remained identical — only the sleep timer was extended. This maintained security posture while accommodating legitimate workflow requirements.

### Challenge: Docking Station Lid-Close Expectations

**Problem:** Users with docking stations expected to close their laptop lid and continue working on external displays. The default Windows lid-close behaviour (sleep) disrupted this workflow prior to policy deployment.

**Solution:** The "Do Nothing" lid-close policy resolved this natively. Post-deployment, docked users could close the lid without triggering any power state change — a significant usability improvement that was communicated as a positive outcome of the policy rollout.

---

## Best Practices / Recommendations

- **Consolidate into a single profile** wherever possible. Multiple overlapping power profiles increase the risk of setting conflicts and complicate troubleshooting. The Settings Catalog's conflict detection works best when all power settings live in one profile.

- **Use Entra ID security groups for targeting** rather than "All Devices." Group-based targeting enables staged rollout, exception management (e.g., power-user groups with extended timers), and clean rollback if issues arise.

- **Tune timeout values to organisational risk tolerance.** The 300/600 second values in this project represent a strong general-purpose baseline. High-security environments should consider reducing both values. Environments with significant idle-but-active workloads may need to extend the sleep timer for specific user groups.

- **Always pair display timeout with authentication-on-resume.** Display timeout without credential enforcement is a power-saving feature, not a security control. These settings are only effective as a security measure when deployed together.

- **Audit for GPO conflicts before deployment.** Hybrid Entra ID joined devices can receive settings from both Intune and on-premises Group Policy. Run `gpresult /h` on pilot devices and use Intune's conflict detection to identify overlapping configurations before estate-wide rollout.

- **Document exception groups.** If certain roles require modified power settings, maintain a clear record of which Entra ID groups receive alternative profiles and the business justification for each exception. This is critical for audit readiness.

- **Monitor deployment health continuously.** Set up an Intune compliance policy or reporting dashboard that alerts on devices where the power management profile has failed to apply. A policy that exists but isn't enforced provides no protection.

---

## Conclusion

This project demonstrates how a single, well-architected Intune configuration profile can enforce comprehensive power management and lockscreen security across an enterprise Windows fleet. By consolidating display timeout, sleep orchestration, physical button suppression, hard disk persistence, and authentication-on-resume into one deployable unit, the solution delivers consistent endpoint security without fragmenting the management surface.

The policy follows a defence-in-depth approach to idle device security — the display timeout reduces visual exposure, the sleep timer escalates to system-level protection, physical buttons are neutralised to prevent bypass, disk availability is maintained for background services, and authentication on resume ensures every wake event requires credential verification. Together, these settings create a closed-loop idle security model that operates without user intervention and satisfies common enterprise compliance frameworks.

---

## Author

**Olu Adelokiki**
Lead EUC Engineer, UserCompute
Website: [www.usercompute.com](https://www.usercompute.com)
