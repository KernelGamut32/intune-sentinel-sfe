# Lab 4: Reading Deployment Logs for Intune Win32 App Troubleshooting

## Lab duration

90 minutes

## Lab purpose

This lab develops student confidence in reading **deployment logs** and forming a support-quality root cause statement. Students will use Intune client logs from a lab device and, if the instructor chooses, additional prepared log excerpts that simulate common Airlock-style failure patterns.

This lab maps directly to the Day 1 content on:

- IME log interpretation
- AgentExecutor log interpretation
- distinguishing client-side and service-side issues
- reading failure patterns instead of just reading lines

---

## Why this lab works in a Microsoft 365 Developer Tenant

The log locations and troubleshooting workflow are standard for Intune Win32 app deployments. Students can generate their own logs from the earlier labs in the same developer tenant, then analyze them. No unsupported tenant configuration is required.

Microsoft Learn confirms that:

- Win32 app client logs are commonly stored under `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs`
- Intune provides troubleshooting details in the admin center
- IME is the component responsible for Win32 app processing on the endpoint

---

## Learning objectives

By the end of the lab, students will be able to:

1. Locate the most relevant logs for Win32 app troubleshooting.
2. Identify where installation starts in the IME log.
3. Recognize download success, install attempts, detection results, and exit-code patterns.
4. Correlate portal status to client-side log evidence.
5. Write a short root-cause statement supported by evidence.

---

## Student access required in the tenant

### Recommended

- read access to Intune app and device status
- ability to sign in to the test device
- local access to the device's log files

### Helpful

- **Read Only Operator** for portal review if students are not modifying apps
- instructor-provided anonymized excerpts for additional pattern practice

---

## Lab environment and materials

Use the same lab device from Labs 1-3.

Primary logs:

- `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log`
- `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\AgentExecutor.log`

Optional additional source:

- exported or instructor-prepared text excerpts from known scenarios

Tools:

- Notepad
- CMTrace if available
- Intune admin center

---

## Lab scenario

You are the support engineer. A customer tells you one of the following:

- the app never installed
- the app is pending or failed
- Intune says installed but the app is not functioning

Your job is to determine:

1. where in the flow the failure occurred
2. what evidence supports that conclusion
3. what the next action should be

---

## Pre-lab setup (10 minutes)

1. Confirm the device is enrolled and online.
2. Open the Intune admin center and locate the device and one of your previous lab apps.
3. On the endpoint, confirm the IME log folder exists.

If the folder does not exist, confirm that at least one Win32 app or PowerShell script has been assigned to the device.

---

## Part 1: Build a log-reading checklist (10 minutes)

Before opening logs, students create a short checklist:

- What app is in scope?
- Is this affecting one device or many?
- What does the Intune portal say?
- Is the problem likely assignment, download, execution, detection, or post-install operation?

This mirrors the Day 1 troubleshooting flow.

---

## Part 2: Find the relevant execution block in IME log (20 minutes)

### Step 1: Open the log

Open:

- `IntuneManagementExtension.log`

### Step 2: Search for the app name

Search for the name of one earlier lab app, such as:

- `Airlock-Style Agent Simulator`
- `Airlock Lab Main Sequenced App`
- `Detection Rule Lab App`

### Step 3: Identify the key milestones

Have students highlight or capture examples of:

- download started or completed
- install started
- exit code
- detection rule result
- retry or timeout indicators

### Step 4: Write a one-sentence status summary

Example format:
> The device downloaded the package and started execution, the installer returned a non-zero exit code, and detection never evaluated true.

---

## Part 3: Compare IME log with AgentExecutor log (20 minutes)

### Step 1: Open `AgentExecutor.log`

Search for the same app or script execution timeframe.

### Step 2: Look for script-specific evidence

Students should identify:

- script start
- script output if present
- exit code
- any obvious PowerShell error text

### Step 3: Compare the logs

Prompt students to answer:

- What did IME conclude?
- What did the script actually do?
- Did the script fail clearly, or did it return success with misleading behavior?

### Step 4: Write a root-cause statement

Example format:
> The dependency script executed but did not create the expected file, causing the main installer to fail. The failure is client-side and occurs during prerequisite sequencing, not during assignment.

---

## Part 4: Analyze four common support patterns (25 minutes)

Use logs generated in prior labs and, if needed, instructor-provided excerpts.

### Pattern A: Installer ran, detection false

Symptoms:

- install appears to complete
- detection rule result is false
- app remains not installed or retries later

Student task:

- identify the detection mismatch
- recommend the corrected detection rule

### Pattern B: Dependency problem

Symptoms:

- main app fails
- dependency evidence is missing or incomplete

Student task:

- identify the prerequisite issue
- determine whether the dependency relationship or dependency script is the likely fault

### Pattern C: Silent success masking a bad outcome

Symptoms:

- exit code is 0
- expected artifact is missing or incomplete
- later steps fail

Student task:

- explain why exit code alone is not enough
- identify what additional evidence is needed

### Pattern D: No execution attempt yet

Symptoms:

- app assigned in Intune
- no relevant install attempt in IME log
- device check-in timing may be the issue

Student task:

- explain why this may be a timing or sync problem rather than an installer failure
- identify what to check next in the portal and on the device

---

## Part 5: Portal plus log correlation exercise (15 minutes)

For one app and one device, have students create a small evidence table with three columns:

- **Portal observation**
- **Device log evidence**
- **Conclusion**

Example rows:

- Pending in portal | no execution block yet in IME log | likely no recent sync or processing yet
- Failed in portal | non-zero exit code in IME log | install execution failure
- Installed in portal | detection true but app not functioning | operational problem may be beyond Intune detection

---

## Expected results

Students should leave the lab able to:

- locate the right Intune client logs quickly
- find the relevant execution block
- interpret common status patterns
- avoid over-trusting portal status alone
- produce a concise, evidence-based next step

---

## Troubleshooting prompts

1. Did the device ever receive and process the assignment?
2. Did the package download?
3. Did the install start?
4. What was the exit code?
5. Did detection evaluate true or false?
6. Is the issue installation, detection, or post-install operation?

---

## Debrief questions

1. Which log is usually your first stop for Win32 app issues?
2. When does `AgentExecutor.log` become especially valuable?
3. What is the danger of treating `Installed` in Intune as proof that the app is healthy?
4. What evidence would you request from a customer if the local logs are not enough?
5. How would you distinguish a tenant-side configuration issue from a device-side execution issue?

---

## Cleanup

No special cleanup is required unless you want to remove prior lab apps.

---

## Instructor notes

This lab works best after the first three labs because students can analyze logs from actions they actually performed. If time is short, make Part 4 an instructor-led exercise using prepared excerpts and spend more time on Parts 2 and 3.

---

## Authoritative references

- Microsoft Learn: **Troubleshoot Win32 apps in Microsoft Intune**
- Microsoft Learn: **Understand Microsoft Intune Management Extension**
- Microsoft Learn: **Add, assign, and monitor a Win32 app in Microsoft Intune**
