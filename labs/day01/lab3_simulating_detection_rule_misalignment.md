# Lab 3: Simulating Detection Rule Misalignment in Microsoft Intune

## Lab duration

90 minutes

## Lab purpose

This lab helps students experience one of the most common support situations from Day 1: **the install behavior and the detection rule do not agree**. Students will deliberately configure incorrect detection logic, observe how Intune behaves, and then correct the mismatch.

This lab focuses on:

- file detection mismatch
- stale version logic
- registry redirection awareness
- why `Installed` or `Not installed` in Intune can be misleading if the detection rule is wrong

---

## Why this lab works in a Microsoft 365 Developer Tenant

The exercise uses a simulated app and standard Intune Win32 detection capabilities. No production software is needed. All steps are compatible with a Microsoft 365 Developer Tenant as long as students have an enrolled Windows test device.

Microsoft Learn confirms that:

- Win32 detection can be configured manually or by custom script
- all configured detection conditions must be met
- file, registry, and MSI detection each have specific behaviors
- Intune can re-offer required apps when detection determines the app is not present

---

## Learning objectives

By the end of this lab, students will be able to:

1. Create a Win32 app whose install outcome is easy to verify.
2. Configure a detection rule that is intentionally wrong.
3. Observe how Intune reacts to false-negative detection.
4. Explain how wrong paths, version drift, or wrong registry location create repeated reinstall attempts or bad status reporting.
5. Correct the detection rule and verify recovery.

---

## Student access required in the tenant

### Recommended

- **Intune Application Manager** or equivalent permissions to create and edit apps and assignments
- read access to device and install status
- enrolled Windows test device or VM

### Optional

- instructor-provided prebuilt package to save time

---

## Lab environment and materials

Students create a simple test app that writes:

- a file to `C:\Program Files\DetectionRuleLab\app.exe`
- a version marker in the registry at `HKLM\SOFTWARE\DetectionRuleLab`

Local folders:

- `C:\Lab\DetectionRuleLab\Source`
- `C:\Lab\DetectionRuleLab\Output`

---

## Lab scenario

A support engineer receives a ticket saying the app is either:

- installing repeatedly, or
- showing the wrong status in Intune

The root cause is not the installer. The root cause is the **detection rule**.

You will create one app and then test multiple detection mistakes against it.

---

## Pre-lab setup (10 minutes)

1. Confirm the Windows device is enrolled.
2. Confirm the device is in `Lab-Intune-Win32-Devices`.
3. Remove prior lab artifacts if needed:

```powershell
Remove-Item 'C:\Program Files\DetectionRuleLab' -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item 'HKLM:\SOFTWARE\DetectionRuleLab' -Recurse -Force -ErrorAction SilentlyContinue
```

---

## Part 1: Build the test app (20 minutes)

### Step 1: Create the install script

Create `install.ps1` in `C:\Lab\DetectionRuleLab\Source`:

```powershell
$installRoot = 'C:\Program Files\DetectionRuleLab'
$regPath = 'HKLM:\SOFTWARE\DetectionRuleLab'

New-Item -Path $installRoot -ItemType Directory -Force | Out-Null
'fake binary' | Out-File "$installRoot\app.exe" -Force

New-Item -Path $regPath -Force | Out-Null
New-ItemProperty -Path $regPath -Name 'Version' -Value '1.0.0.1' -PropertyType String -Force | Out-Null
exit 0
```

### Step 2: Create the uninstall script

Create `uninstall.ps1`:

```powershell
Remove-Item 'C:\Program Files\DetectionRuleLab' -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item 'HKLM:\SOFTWARE\DetectionRuleLab' -Recurse -Force -ErrorAction SilentlyContinue
exit 0
```

### Step 3: Package the app

Create the `.intunewin` package.

### Step 4: Add the Win32 app to Intune

Use:

- **Name**: `Detection Rule Lab App`
- **Install command**:

  ```powershell
  "%WINDIR%\sysnative\windowspowershell\v1.0\powershell.exe" -ExecutionPolicy Bypass -File .\install.ps1
  ```

- **Uninstall command**:

  ```powershell
  "%WINDIR%\sysnative\windowspowershell\v1.0\powershell.exe" -ExecutionPolicy Bypass -File .\uninstall.ps1
  ```

- **Install behavior**: `System`

Assign it as **Required** to `Lab-Intune-Win32-Devices`.

---

## Part 2: Scenario A - Wrong file path (20 minutes)

### Step 1: Configure the wrong detection rule

Use file detection with:

- **Path**: `C:\Program Files\TestApp`
- **File**: `app.exe`

But the installer actually writes to:

- `C:\Program Files\DetectionRuleLab\app.exe`

### Step 2: Sync and observe

Force a sync and wait for processing.

### Step 3: Verify the real install state locally

Confirm on the endpoint:

- `C:\Program Files\DetectionRuleLab\app.exe` exists

### Step 4: Check Intune status and logs

Review:

- install status in the portal
- `IntuneManagementExtension.log`

Expected result:

- install command succeeds
- detection evaluates false
- Intune does not consider the app installed
- required assignment may lead to repeated retry/re-offer behavior over time

### Step 5: Fix it

Correct the detection rule to:

- **Path**: `C:\Program Files\DetectionRuleLab`
- **File**: `app.exe`

Sync again and verify the app becomes correctly detected.

---

## Part 3: Scenario B - Stale version rule (20 minutes)

### Step 1: Change detection to registry-based version validation

Create a registry detection rule that checks:

- key path: `HKLM\SOFTWARE\DetectionRuleLab`
- value name: `Version`
- detection method expecting `1.0.0.0`

But the app actually writes `1.0.0.1`.

### Step 2: Observe behavior

Expected result:

- install succeeds
- detection still fails because expected version is stale
- students see how successful upgrades can be misreported when detection logic is not updated

### Step 3: Fix the version expectation

Update the rule to expect `1.0.0.1`.

Re-sync and confirm resolution.

---

## Part 4: Scenario C - Registry location discussion and optional demo (10 minutes)

### Safe classroom approach

Because building a clean 32-bit registry redirection demo can take extra time, use this as either:

- an instructor demo, or
- a guided discussion with screenshots and registry examples

Discuss:

- installer writes to `HKLM\SOFTWARE\WOW6432Node\...`
- detection incorrectly points to `HKLM\SOFTWARE\...`
- Intune evaluates the wrong path and reports failure or retries

If the instructor wants to extend the lab, create a second installer script that writes intentionally to a WOW6432Node path and then repeat the detection exercise.

---

## Expected results

Students should be able to explain that:

- detection is evaluated after installation
- detection must match the actual post-install state
- incorrect detection rules can make a good install look bad
- a stale detection rule can trigger repeat install attempts or false status

---

## Troubleshooting prompts

1. Does the installed file really exist where the detection rule expects it?
2. Is the registry key path correct?
3. Does the value type and value content match exactly?
4. Is the app 32-bit or 64-bit, and does that affect the registry path?
5. Is the detection checking what matters for support, or just what is easy to configure?

---

## Debrief questions

1. Why can a detection rule problem look like an installer problem?
2. What support symptoms would a stale detection rule create after an upgrade?
3. Why is confirming the actual post-install state on a known-good machine so important before defining detection?
4. Which is usually more reliable for a security agent: file presence, registry value, or custom script detection?
5. When would custom script detection be worth the extra complexity?

---

## Cleanup

1. Remove the assignment.
2. Delete or uninstall the app.
3. Remove local files and registry artifacts.

---

## Authoritative references

- Microsoft Learn: **Add, assign, and monitor a Win32 app in Microsoft Intune**
- Microsoft Learn: **Troubleshoot Win32 apps in Microsoft Intune**
- Microsoft Learn: **Understand Microsoft Intune Management Extension**
