# Lab 2: Reviewing Script Sequencing Behavior in Microsoft Intune

## Lab duration

90 minutes

## Lab purpose

This lab teaches students how **sequencing** works for Win32 app dependencies in Intune and why sequencing errors are such a common cause of Airlock deployment failures. Students will deploy a cleanup step as a Win32 dependency, deploy a main app that expects the cleanup result, and then compare correct and incorrect sequencing behavior.

This directly reinforces the Day 1 material on:

- dependencies and sequencing
- exit codes
- IME and AgentExecutor log analysis
- why Platform Scripts are not a substitute for Win32 dependencies when execution order matters

---

## Why this lab works in a Microsoft 365 Developer Tenant

This lab uses only Intune-supported Win32 app functionality and PowerShell-based simulation. No production agent is required. The exercise is fully achievable in a Microsoft 365 Developer Tenant as long as students have an enrolled Windows device or VM.

Microsoft Learn confirms that:

- Win32 app dependencies can be configured so required dependent apps install before the main app
- dependent apps can be automatically installed even when not directly targeted
- IME is the client-side execution component for Win32 apps and scripts assigned through Intune

---

## Learning objectives

By the end of the lab, students will be able to:

1. Build a two-step dependency chain in Intune.
2. Explain how automatic dependency installation works.
3. Observe how the main app behaves when a prerequisite is missing.
4. Compare success, failure, and misleading success conditions.
5. Use IME logs to identify which step actually failed.

---

## Student access required in the tenant

### Recommended

- **Intune Application Manager** role, or equivalent permissions to create and assign apps
- ability to view device status
- a licensed user and enrolled Windows test device

### Helpful but optional

- **Read Only Operator** or broader read access for students who only need to review results
- instructor-created device group and pre-enrolled VM

---

## Lab environment and materials

Use the same enrolled Windows device or VM from Lab 1, or a fresh one.

Students will create three local source folders:

- `C:\Lab\Sequencing\CleanupApp`
- `C:\Lab\Sequencing\MainApp`
- `C:\Lab\Sequencing\Output`

The lab uses a flag file at:

- `C:\ProgramData\AirlockStyleLab\cleanup_done.txt`

---

## Lab scenario

A customer deployment requires a cleanup script to run before a main security agent installer. The main installer expects a known-good environment marker. In this simulation:

- the cleanup dependency creates the marker file
- the main installer checks for the marker file
- if the marker is missing, the main installer fails

You will first make the sequence work correctly, then intentionally break part of the logic and inspect the logs.

---

## Pre-lab setup (10 minutes)

1. Confirm the device is enrolled and visible in Intune.
2. Confirm you can still sync the device manually.
3. Create or reuse the device group `Lab-Intune-Win32-Devices`.

Optional cleanup before starting:

```powershell
Remove-Item 'C:\ProgramData\AirlockStyleLab' -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item 'C:\Program Files\AirlockSequencedApp' -Recurse -Force -ErrorAction SilentlyContinue
```

---

## Part 1: Build the cleanup dependency app (20 minutes)

### Step 1: Create the cleanup source folder

Create:

- `C:\Lab\Sequencing\CleanupApp`

### Step 2: Create the cleanup script

Create `cleanup.ps1` with this content:

```powershell
$root = 'C:\ProgramData\AirlockStyleLab'
New-Item -Path $root -ItemType Directory -Force | Out-Null
'cleanup complete' | Out-File "$root\cleanup_done.txt" -Force
exit 0
```

### Step 3: Create a simple uninstall script

Create `cleanup-uninstall.ps1`:

```powershell
Remove-Item 'C:\ProgramData\AirlockStyleLab\cleanup_done.txt' -Force -ErrorAction SilentlyContinue
exit 0
```

### Step 4: Package the cleanup app

Use `IntuneWinAppUtil.exe` to create an `.intunewin` package for the cleanup app.

### Step 5: Create the cleanup Win32 app in Intune

Use:

- **Name**: `Airlock Lab Cleanup Dependency`
- **Install command**:

  ```powershell
  powershell.exe -ExecutionPolicy Bypass -File .\cleanup.ps1
  ```

- **Uninstall command**:

  ```powershell
  powershell.exe -ExecutionPolicy Bypass -File .\cleanup-uninstall.ps1
  ```

- **Install behavior**: `System`

Detection rule:

- **File exists**
- Path: `C:\ProgramData\AirlockStyleLab`
- File: `cleanup_done.txt`

Do **not** directly assign this app yet.

---

## Part 2: Build the main app (20 minutes)

### Step 1: Create the main app source folder

Create:

- `C:\Lab\Sequencing\MainApp`

### Step 2: Create the main installer script

Create `install-main.ps1`:

```powershell
$flag = 'C:\ProgramData\AirlockStyleLab\cleanup_done.txt'
$installRoot = 'C:\Program Files\AirlockSequencedApp'

if (-not (Test-Path $flag)) {
    Write-Host 'Cleanup flag missing. Failing install.'
    exit 1
}

New-Item -Path $installRoot -ItemType Directory -Force | Out-Null
'Sequenced install complete' | Out-File "$installRoot\installed.txt" -Force
exit 0
```

### Step 3: Create the uninstall script

Create `uninstall-main.ps1`:

```powershell
Remove-Item 'C:\Program Files\AirlockSequencedApp' -Recurse -Force -ErrorAction SilentlyContinue
exit 0
```

### Step 4: Package the main app

Package the folder with `IntuneWinAppUtil.exe`.

### Step 5: Create the main Win32 app in Intune

Use:

- **Name**: `Airlock Lab Main Sequenced App`
- **Install command**:

  ```powershell
  powershell.exe -ExecutionPolicy Bypass -File .\install-main.ps1
  ```

- **Uninstall command**:

  ```powershell
  powershell.exe -ExecutionPolicy Bypass -File .\uninstall-main.ps1
  ```

- **Install behavior**: `System`

Detection rule:

- File exists
- Path: `C:\Program Files\AirlockSequencedApp`
- File: `installed.txt`

---

## Part 3: Configure dependency sequencing (10 minutes)

### Step 1: Add the dependency

In the main app record, open **Dependencies** and add:

- `Airlock Lab Cleanup Dependency`

Set **Automatically install** to `Yes`.

> This is the supported Intune method for ensuring the required app is installed before the main app.

### Step 2: Assign the main app

Assign the **main app only** as:

- **Required**
- to `Lab-Intune-Win32-Devices`

Do not separately assign the cleanup app.

### Step 3: Sync the device

Force a sync on the endpoint.

---

## Part 4: Validate successful sequencing (10 minutes)

### Step 1: Validate local artifacts

On the endpoint confirm:

- `C:\ProgramData\AirlockStyleLab\cleanup_done.txt` exists
- `C:\Program Files\AirlockSequencedApp\installed.txt` exists

### Step 2: Validate app status in Intune

Review installation status for:

- cleanup dependency
- main app

### Step 3: Review logs

Inspect:

- `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log`
- `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\AgentExecutor.log`

Search for:

- dependency installation
- app name
- exit code
- detection results

Ask students:

- Which app executed first?
- How do you know?
- Where is the proof in the log?

---

## Part 5: Break the behavior and compare outcomes (15 minutes)

### Scenario A: Remove the dependency relationship

1. Remove the dependency from the main app.
2. Uninstall or delete the cleanup marker file from the device.
3. Re-sync the device.

Expected result:

- main app attempts installation
- install fails because the marker file is missing
- log shows failure with a non-zero exit code

### Scenario B: Discuss, do not deploy, the unsupported pattern

Review what would happen if the cleanup step were deployed as a **Platform Script** instead of a dependency.

Discussion points:

- no guaranteed ordering relative to the Win32 app
- timing ambiguity
- much harder root-cause analysis when the main installer begins before the prep step is complete

> In a shared tenant, keep this as a discussion or instructor demo rather than creating unnecessary competing assignments.

---

## Expected results

Students should observe that:

- dependencies are enforced before the main app install attempt
- the main app succeeds only when the cleanup marker exists
- the logs clearly show sequencing and outcome
- removing the dependency causes the main app to fail in a predictable way

---

## Troubleshooting prompts

1. Is **Automatically install** set to `Yes` for the dependency?
2. Does the dependency app have a working detection rule?
3. Did the cleanup script actually create the flag file?
4. Is the main app assigned to a device group as **Required**?
5. Do logs show the dependency completed before the main app started?

---

## Debrief questions

1. Why are Win32 dependencies the correct sequencing mechanism for Airlock-style deployments?
2. Why is direct assignment of the dependency often unnecessary?
3. What is the support risk of using separately targeted Platform Scripts for prerequisite steps?
4. Which log tells the story more clearly here: `IntuneManagementExtension.log` or `AgentExecutor.log`?
5. What customer symptoms might this create in a real Airlock deployment?

---

## Cleanup

1. Remove the assignment for the main app.
2. Delete or uninstall both lab apps.
3. Remove local folders if desired.

---

## Authoritative references

- Microsoft Learn: **Add, assign, and monitor a Win32 app in Microsoft Intune**
- Microsoft Learn: **Understand Microsoft Intune Management Extension**
- Microsoft Learn: **Troubleshoot Win32 apps in Microsoft Intune**
