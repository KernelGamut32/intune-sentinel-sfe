# Lab 1: Packaging and Deploying an Airlock-Style Win32 App in Microsoft Intune

## Lab duration

90 minutes

## Lab purpose

This lab gives students a safe, repeatable way to practice the same **Win32 packaging and deployment workflow** used for Airlock-style agent deployments, without needing a production security agent. Students will package a small simulated endpoint agent, upload it as a **Win32 app (.intunewin)**, assign it in Intune, deploy it to a test device, and validate that Intune reports it correctly.

This lab is aligned to the Day 1 deck sections on:

- Win32 packaging and the `.intunewin` format
- Intune Management Extension (IME)
- install commands, detection rules, and assignment behavior
- end-to-end deployment flow

> Instructor note: This lab intentionally uses a **simulated Airlock-style app** built from PowerShell and files you create during class. That keeps the lab safe, legal, and fully achievable in a Microsoft 365 Developer Tenant.

---

## Why this lab works in a Microsoft 365 Developer Tenant

A Microsoft 365 E5 developer sandbox includes Microsoft 365, Microsoft Entra ID, and Enterprise Mobility + Security licenses for development use, which makes it suitable for Intune-based lab exercises when you have an enrolled Windows device available for testing. The tenant itself does **not** provide a Windows endpoint, so each student or pair needs access to a Windows 10/11 device or VM that can be enrolled into Intune for the exercise.

Microsoft Learn confirms that:

- the Microsoft 365 E5 developer sandbox includes Enterprise Mobility + Security and test users
- Win32 app management requires supported Windows devices that are joined or registered to Microsoft Entra ID and enrolled in Intune
- the Intune Management Extension installs automatically when a Win32 app or PowerShell script is assigned

---

## Learning objectives

By the end of this lab, students will be able to:

1. Build a source folder for a Win32 app package.
2. Use the Microsoft Win32 Content Prep Tool to create an `.intunewin` package.
3. Create a Win32 app in Intune with a silent install command.
4. Configure a practical file-based detection rule.
5. Assign the app as **Required** to a **device group**.
6. Validate the deployment on the endpoint and in Intune.
7. Correlate portal status with local evidence on the device.

---

## Student access required in the tenant

Students need one of the following access models.

### Recommended for full hands-on execution

- **Intune Application Manager** role, or broader Intune permissions that allow app creation and assignment
- Ability to read device information in Intune
- A licensed test user account for device enrollment
- Permission to enroll a Windows device into Intune

### Optional instructor-only setup

The instructor can perform tenant-wide setup ahead of time:

- create the device group
- confirm automatic enrollment settings
- provide the student with a pre-enrolled Windows VM

### Device requirements

Each student or pair needs:

- Windows 10/11 Pro, Enterprise, or Education
- local administrator rights on the lab device or VM
- internet connectivity to reach Intune service endpoints
- Company Portal installed if you want students to use manual sync from Company Portal

---

## Lab environment and materials

### Required

- Microsoft 365 Developer Tenant with Intune available
- One enrolled Windows test device or VM per student or pair
- Intune admin center access
- Microsoft Win32 Content Prep Tool (`IntuneWinAppUtil.exe`)

### Student-created files for this lab

Students will create these files locally in `C:\Lab\AirlockStyleAgent`:

- `install-agent.ps1`
- `uninstall-agent.ps1`
- `detect-agent.ps1` (optional validation helper)
- `agent.bin` (empty placeholder file)

---

## Lab scenario

Your customer wants to deploy an endpoint security agent through Intune. For training, you will simulate that agent by:

- creating a folder under `C:\Program Files\AirlockStyleAgent`
- copying a placeholder binary file into that folder
- creating a registry key under `HKLM\SOFTWARE\AirlockStyleAgent`
- writing an install log file

This behaves enough like a real agent deployment to teach packaging, assignment, detection, and validation without using a live security product.

---

## Pre-lab setup (10 minutes)

### Step 1: Confirm device enrollment

On the Windows lab device:

1. Open **Settings > Accounts > Access work or school**.
2. Confirm the device is connected to the developer tenant.
3. In the Intune admin center, confirm the device appears under **Devices > Windows**.
4. Record the device name.

### Step 2: Confirm IME can be used

IME installs automatically when a Win32 app or PowerShell script is assigned, but the device must be enrolled and on a supported Windows edition.

Instructor check:

- device is visible in Intune
- device is online
- device is a supported edition

### Step 3: Create or confirm the target device group

In Intune or Entra:

1. Create a **security group** named `Lab-Intune-Win32-Devices` if it does not already exist.
2. Add the student lab device to the group.

> Use a **device group**, not a user group. That matches Day 1 guidance for system-level security software.

---

## Part 1: Create the simulated agent source folder (15 minutes)

### Step 1: Create the folder structure

On the lab admin workstation, create:

- `C:\Lab\AirlockStyleAgent\Source`
- `C:\Lab\AirlockStyleAgent\Output`

### Step 2: Create the placeholder binary

Inside `C:\Lab\AirlockStyleAgent\Source`, create an empty file named `agent.bin`.

PowerShell example:

```powershell
New-Item -Path 'C:\Lab\AirlockStyleAgent\Source\agent.bin' -ItemType File -Force
```

### Step 3: Create the installer script

Create `install-agent.ps1` in the `Source` folder with this content:

```powershell
$installRoot = 'C:\Program Files\AirlockStyleAgent'
$regPath = 'HKLM:\SOFTWARE\AirlockStyleAgent'
$logPath = 'C:\ProgramData\AirlockStyleAgent'

New-Item -Path $installRoot -ItemType Directory -Force | Out-Null
New-Item -Path $logPath -ItemType Directory -Force | Out-Null
Copy-Item -Path '.\agent.bin' -Destination "$installRoot\agent.bin" -Force

New-Item -Path $regPath -Force | Out-Null
New-ItemProperty -Path $regPath -Name 'Version' -Value '1.0.0.0' -PropertyType String -Force | Out-Null
New-ItemProperty -Path $regPath -Name 'InstalledBy' -Value 'IntuneWin32Lab' -PropertyType String -Force | Out-Null

'Install completed' | Out-File "$logPath\install.log" -Force
exit 0
```

### Step 4: Create the uninstall script

Create `uninstall-agent.ps1`:

```powershell
$installRoot = 'C:\Program Files\AirlockStyleAgent'
$regPath = 'HKLM:\SOFTWARE\AirlockStyleAgent'
$logPath = 'C:\ProgramData\AirlockStyleAgent'

Remove-Item -Path "$installRoot\agent.bin" -Force -ErrorAction SilentlyContinue
Remove-Item -Path $installRoot -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path $regPath -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path $logPath -Recurse -Force -ErrorAction SilentlyContinue

exit 0
```

### Step 5: Test locally

From an elevated PowerShell session, run:

```powershell
Set-Location 'C:\Lab\AirlockStyleAgent\Source'
powershell.exe -ExecutionPolicy Bypass -File .\install-agent.ps1
```

Verify:

- `C:\Program Files\AirlockStyleAgent\agent.bin` exists
- registry key `HKLM\SOFTWARE\AirlockStyleAgent` exists
- `C:\ProgramData\AirlockStyleAgent\install.log` exists

Then run the uninstall script and confirm cleanup worked.

> This local test is important. A Win32 package should always be validated before upload.

---

## Part 2: Package the Win32 app (15 minutes)

### Step 1: Run the Win32 Content Prep Tool

Run `IntuneWinAppUtil.exe` and provide:

- **Source folder**: `C:\Lab\AirlockStyleAgent\Source`
- **Setup file**: `install-agent.ps1` is not a binary, so create a launcher command by packaging the folder and using PowerShell as the Intune install command later
- **Output folder**: `C:\Lab\AirlockStyleAgent\Output`

> The tool packages the source folder into a single encrypted `.intunewin` file for Intune upload.

### Step 2: Confirm package creation

Verify that an `.intunewin` file exists in the output folder.

Record:

- file name
- package creation time

---

## Part 3: Create the Win32 app in Intune (20 minutes)

### Step 1: Add the app

In the Intune admin center:

1. Go to **Apps > Windows > Add**.
2. Select **Windows app (Win32)**.
3. Upload the generated `.intunewin` file.

### Step 2: Configure app information

Use values such as:

- **Name**: `Airlock-Style Agent Simulator`
- **Publisher**: `Training Lab`
- **Category**: `Security`
- **Description**: `Simulated endpoint agent used for Intune packaging and detection labs.`

### Step 3: Configure program settings

Use:

- **Install command**

  ```powershell
  powershell.exe -ExecutionPolicy Bypass -File .\install-agent.ps1
  ```

- **Uninstall command**

  ```powershell
  powershell.exe -ExecutionPolicy Bypass -File .\uninstall-agent.ps1
  ```

- **Install behavior**: `System`
- **Device restart behavior**: determine behavior based on return codes, or no specific action
- **Installation time required**: keep the default unless your instructor specifies otherwise

### Step 4: Requirements

Configure a minimum Windows version appropriate for your lab.
Leave advanced requirements empty unless the instructor wants to extend the lab.

### Step 5: Detection rule

Choose **manually configure detection rules** and create a **file exists** rule:

- **Path**: `C:\Program Files\AirlockStyleAgent`
- **File or folder**: `agent.bin`
- **Detection method**: `File or folder exists`
- **Associated with a 32-bit app on 64-bit clients**: `No`

> This detection rule is intentionally simple and easy to validate.

### Step 6: Assignments

Assign the app as:

- **Required**
- to the **Lab-Intune-Win32-Devices** device group

Finish the wizard.

---

## Part 4: Trigger deployment and validate (20 minutes)

### Step 1: Force a device sync

On the target device, use one of these methods:

- Company Portal sync
- Settings > Accounts > Access work or school > Info > Sync

### Step 2: Watch for deployment

Allow time for:

- assignment propagation
- device sync
- IME download and execution
- detection and reporting

### Step 3: Validate locally on the device

Check:

- `C:\Program Files\AirlockStyleAgent\agent.bin`
- `C:\ProgramData\AirlockStyleAgent\install.log`
- registry key `HKLM\SOFTWARE\AirlockStyleAgent`

### Step 4: Validate in Intune

In the app record, review:

- device install status
- installed device count
- any failures or pending states

### Step 5: Validate IME logging

Open:

- `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log`

Search for:

- app name
- install start
- download success
- detection evaluation

---

## Expected results

At the end of the lab:

- the app is packaged as `.intunewin`
- the app is uploaded to Intune
- the device receives the deployment
- the install command runs under SYSTEM context
- the file-based detection rule evaluates to true
- Intune shows the app as installed for the target device

---

## Troubleshooting prompts for students

If the app does not install, check these first:

1. Was the app assigned to a **device group**?
2. Was the assignment **Required**?
3. Is the device enrolled and recently checked in?
4. Is the install command correct?
5. Does the detection rule match the actual installed path?
6. Does IME log show the app was downloaded and executed?

---

## Debrief questions

1. Why is Win32 packaging preferred for Airlock-style deployments with multiple files or scripts?
2. Why is **System** install behavior the better choice here?
3. What would happen if you assigned this app to a **user group** instead?
4. Why can Intune show `Installed` even when a real security agent is not fully operational?
5. Which evidence do you trust most during troubleshooting: portal status, local files, registry, or IME logs?

---

## Stretch exercise (optional)

Modify the installer so it also creates a Windows service placeholder log or writes a version file. Then update the detection rule to use:

- registry detection, or
- file version detection, or
- a custom detection script

---

## Cleanup

1. Remove the Intune assignment.
2. Uninstall the app from the test device if desired.
3. Delete the app from Intune after the lab.
4. Remove the local lab folder `C:\Lab\AirlockStyleAgent`.

---

## Instructor notes

- This lab is intentionally conservative and tenant-safe.
- It avoids unsupported or risky actions in a shared developer tenant.
- It gives students direct practice with the same Intune workflow emphasized in the Day 1 deck: source folder, package, upload, assignment, device sync, IME execution, and detection.

---

## Authoritative references

- Microsoft Learn: **Set up a Microsoft 365 developer sandbox subscription**
- Microsoft Learn: **Prepare a Win32 app to be uploaded to Microsoft Intune**
- Microsoft Learn: **Add, assign, and monitor a Win32 app in Microsoft Intune**
- Microsoft Learn: **Understand Microsoft Intune Management Extension**
- Microsoft Learn: **Troubleshoot Win32 apps in Microsoft Intune**
