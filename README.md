# Windows Signal
Windows Signal is a repository that contains the SIOS iQ integration with Windows Events that can be sent to SIOS iQ product for analysis and correlation.

# References
Windows Signal depends on Signal iQ SDK (https://github.com/siostechcorp/Signal_iQ).

# Prerequisites
This module depends on Python 2.7.14, and certain packages as outlined in the Signal iQ SDK. See the Signal iQ start up guide for setup instructions prior to using Windows Signal.

# Instructions
Window Signal consists of two parts: (1) the scripts included in this repo, and (2) local system task configuration. A recurring task must be configured to run on some time interval. This task simply needs to call the included PowerShell script.

The included PowerShell script can be edited with your iQ Environment id or can be passed in via the CLI. The Environment id should have been obtained during the Signal iQ setup performed as a prerequisite, but it is also availble in the Properties page for your environment in the SIOS iQ web gui.

The PowerShell script relies on a json file declaring which events are of interest and should be monitored. An example file (Windows.json) can be found in the json folder, and a sample file (Sample.json) with syntax descriptions is in the base folder. Additional events can be monitored by adding to the Windows.json file or by adding another file in the json folder containing the desired events using the same format as the existing files. The basic layout for the json structure is Log > Source > ID > FilterText. The FilterText can be used to narrow down a single event from multiple different events using the same (Log,Source,ID). Use a '.' in place of the FilterText to match any(all) events with the same (Log,Source,ID).

To create a task in Windows Server first start the Task Scheduler (https://technet.microsoft.com/en-us/library/cc721931(v=ws.11).aspx)  

Next schedule a task (https://technet.microsoft.com/en-us/library/cc748993(v=ws.11).aspx) similar to the images below.  
The arguments field on the New Action panel should be given the following:  

"<path-to-Windows_Signal-repo>\scripts\Send-Signal.ps1"  

Also, the polling interval should match the value used in the "Repeat task every" field in the New Trigger dialog.  

Step by step-
How to install and configure Windows_Signal:
  1. Install and configure Signal_iQ.
  2. Download Windows_Signal repo contents and extract.
  3. Copy report_event.py located in the Windows_Signal directory into the Signal_iQ\python\test\scripts directory.

Test configuration:
  1. Open Powershell
  2. Change directories to the Windows_Signal directory.
  3. Run this command>> `C:\[path to Windows_Signal directory]\Send-Signal.ps1 -PyModule "C:\[path to Signal_iQ directory]\python\test\scripts\report_event.py" -EventsJsonFilePath "C:\[path to Windows_Signal directory]\json" -EnvironmentID [envID]`
  4. For more detailed output, use the -Verbose flag at the end of the previous step.
       
Create a scheduled task:
  1. Open the Windows Task Scheduler
  2. Create task
  3. [General tab] Name- SIOS Signal; Security Options- Run whether user is logged on or not, Run with highest privileges
  4. [Triggers tab] New trigger- Begin the task As startup, New trigger- On a schedule, Daily; advanced settings, repeat task every 5 minutes for a duration of indefinitely.
  5. [Actions tab] New action- Action, start a program; Program/script, Powershell.exe; Add arguments, `C:\[path to Windows_Signal directory]\Send-Signal.ps1 -PyModule "C:\[path to Signal_iQ directory]\python\test\scripts\report_event.py" -EventsJsonFilePath "C:\[path to Windows_Signal directory]\json" -EnvironmentID [envID]`
  6. [Conditions tab] Enable Wake the computer to run this task
  7. [Settings tab] Enable Allow task to be run on demand; Disable Stop the task if it runs longer than
  8. Click OK on the task and enter administrator credentials.
    
# Example Installation 
Below video provides walkthrough of Signal iQ configuration on Windows Server 2016:

[![IMAGE ALT TEXT](https://i.ytimg.com/vi/aag-5UH-UNM/hqdefault.jpg)](https://youtu.be/aag-5UH-UNM "Signal iQ Configuration")

Click the link (and then 'View Raw')for a video configuration walkthrough of Windows Signal:

[Windows Signal Configuration](../master/Windows_Signal.webm)

The following PowerShell code will configure a Task after prompting for all the required inputs:

```
# prompt user for important paths
$psscriptpath = Read-Host "Enter the directory containing the Send-Signal.ps1 file"
$pypath = Read-Host "Enter the directory containing the report_event.py file"
$jsonpath = Read-Host "Enter the directory containing the event json file(s)"
$environmentID =  Read-Host "Enter your iQ environment ID (format: 123456789)"

# prompt user for (domain) admin credentials for creating new task
$message = "Enter administrator credentials to create and run a new Task. Domain administrator recommended."
$credential = $Host.UI.PromptForCredential("Administrator Credentials",$message,"$env:userdomain\$env:username",$env:userdomain)
$username = $credential.UserName
$password = $credential.GetNetworkCredential().Password

# new scheduled task properties to run the ps script every 5 minutes after boot
$action = New-ScheduledTaskAction -Execute "Powershell.exe" -Argument "'$psscriptpath\Send-Signal.ps1' -PyModule '$pypath\report_event.py' -EventsJsonFilePath '$jsonpath' -EnvironmentID $environmentID"
$triggers = [System.Collections.ArrayList]@()
# trigger repeats for 10 years because of differences between how PSv4 and later versions deal with TimeSpan.MaxValue
$triggers.Add((New-ScheduledTaskTrigger -Once -At ([System.DateTime]::Now) -RepetitionDuration (New-TimeSpan -Days 3650) -RepetitionInterval (New-TimeSpan -Minutes 5))) >$Null
$triggers.Add((New-ScheduledTaskTrigger -AtStartup)) >$Null

# create the new task and start it right now
Register-ScheduledTask -Action $action -Trigger $triggers -TaskName "DataKeeper Signal" -Description "Scan for DataKeeper Signal events every 5 minutes; starts on boot." -RunLevel Highest -User $username -Password $password | Start-ScheduledTask
```

