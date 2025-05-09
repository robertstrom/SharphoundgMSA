---
share: "true"
---
  
  
#Bloodhound  
#Sharphound  
#Discovery  
#AttackPaths  
#Docker   
  
## Creation of the Sharphound PowerShell script to be run as a scheduled task  
  
- Now the PowerShell script to run Sharphound via a scheduled task needs to be created  
- Create a directory named Sharphound in the `C:\Support` directory  
- Create a sub-directory named `Results` in the `C:\Support\Sharphound`directory  
- Copy the Sharphound executable to the `C:\Support\Sharphound `directory on the server where the scheduled task will be run (it is recommended that the Sharphound executable be zipped in a password protected zip file so that it does not get prevented by Microsoft Defender during the file transfer)  
- The PowerShell script itself is very simple. The script contains:  
	- The full path to the Sharphound executable  
	- The command line arguments that you want to pass to Sharphound  
	- Redirection of the output to a log file (this was very helpful in getting this setup running initially and may be helpful for troubleshooting in the future)  
	- Deletion of files in the `C:\Support\Sharphound\Results` directory that are older than 2 months  
	- The PowerShell script named `Sharphound_Collection.ps1` should be saved to the `C:\Support\Sharphound` directory  
	- The names of the base --zipfilename and the log file should be modified for each specific domain where this is deployed.  
  
```  
$FileDateStamp = Get-Date -Format FileDateTime  
C:\Support\SharpHound\SharpHound.exe -c all --outputdirectory C:\Support\SharpHound\Results\ --zipfilename <DomainName>_BloodHound_gmsa.zip *> "C:\Support\SharpHound\Results\$FileDateStamp-<DomainName>_sharphound-gmsa.log"  
  
$Folder = "C:\Support\Sharphound\Results"  
  
#Delete files older than 2 months  
Get-ChildItem $Folder -Recurse -Force -ea 0 |  
? {!$_.PsIsContainer -and $_.LastWriteTime -lt (Get-Date).AddDays(-60)} |  
ForEach-Object {  
   $_ | del -Force  
   $_.FullName | Out-File "C:\Support\Sharphound\$FileDateStamp-deletedlog.txt"  
   }  
```  
  
- The resulting directory structure should resemble the screenshot shown below  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250423122920.png)  
  
- The resulting `SharpHound_Collection.ps1` PowerShell script should resemble the screenshot shown below  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250423122708.png)  
  
  
## Creation of the Scheduled Task  
  
- Chose either the PowerShell method or the GUI method to create the scheduled task. There is not need to (you should not) do both methods. The PowerShell method is almost certainly the best / easiest way to create the scheduled task.  
  
### Creating the Scheduled Task via PowerShell  
  
- Run the PowerShell commands shown below (modifying files names and domain names as necessary)  
  
```  
$arg = "-ExecutionPolicy ByPass -NoProfile -File C:\Support\SharpHound\Sharphound_Collection.ps1"  
$ta = New-ScheduledTaskAction -Execute C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe  -Argument $arg  
$tt = New-ScheduledTaskTrigger -At 9:00 -Weekly -DaysOfWeek Monday  
$ap = New-ScheduledTaskPrincipal -UserID 2022TESTING\t0_gMSA_SHS$ -LogonType Password   
Register-ScheduledTask SharpHoundCollection –Action $ta –Trigger $tt –Principal $ap  
```  
  
- Construct the PowerShell commands needed for your environment in preparation for executing them in an elevated PowerShell shell  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250423123315.png)  
  
- Copy and paste the PowerShell commands into the elevated PowerShell shell to create the scheduled task  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250423123437.png)  
  
- The scheduled task has been successfuly created  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250423123531.png)  
  
- After refreshing the Scheduled Tasks MMC you should see the newly created scheduled task that will run as the gMSA account  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250423123552.png)  
  
  
### Creating the Scheduled Task via the GUI  
  
- Open the Scheduled Tasks MSC on the Sharphound server and create a scheduled task. In the example shown below the scheduled task name is **Sharphound**  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321123246.png)  
  
- Configure the scheduled task to run on the desired schedule  
- Configure the scheduled task with the following **Action**  
	- **Program / script:** `powershell.exe`  
	- **Arguments:** `-ExecutionPolicy ByPass -File C:\Support\SharpHound.ps1`  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321123658.png)  
  
- Click the **OK** button on the **Edit Action**  
- Be sure to configure the scheduled task to run as a standard user account at this time  
- Click the **OK** button on the scheduled task to complete the creation  
- Provide the password for the standard user account when prompted and click the **OK** button  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321123944.png)  
  
  
- You should now have a scheduled task to run Sharphound created. We now need to modify the scheduled task to run as the gMSA account  
  
## Modify the scheduled task to run as the gMSA account  
  
- At this point you should see scheduled task setup to run using the user account you previously configured  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321124316.png)  
  
- Running the following command will modify the scheduled task to run as the gMSA account  
	- This solution to assigning the gMSA account to run the scheduled task was found before the complete [PowerShell Scheduled Task creation method]([https://thesleepyadmins.com/2024/02/05/using-group-managed-services-account-with-scheduled-tasks/](https://thesleepyadmins.com/2024/02/05/using-group-managed-services-account-with-scheduled-tasks/)). The `schtasks` command below comes from the [**stackoverflow** post]([https://stackoverflow.com/questions/62699123/deploy-gmsa-account-as-task-scheduler-user-account](https://stackoverflow.com/questions/62699123/deploy-gmsa-account-as-task-scheduler-user-account)) linked in the Resources section of this document  
  
```  
schtasks /change /TN \SharpHound /RU 2022TESTING\t0_gMSA_SHS$ /RP  
```  
  
- An alternate PowerShell method of modifying the scheduled task to run as the gMSA account  
  
```  
New-ScheduledTaskPrincipal -Id SharpHound -UserID Domain\GMServiceAccount$ -LogonType Password -RunLevel Highest  
```  
  
- You should see a message stating that command was successful  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321124510.png)  
  
- If you open the scheduled task you should see that the account that it will run as is the gMSA account  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321124651.png)  
  
  
