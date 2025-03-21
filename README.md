
#Bloodhound
#Sharphound
#Discovery
#AttackPaths
#Docker 

# Resources

[Create a gMSA for use with SharpHound Enterprise](https://support.bloodhoundenterprise.io/hc/en-us/articles/9295298941723-Create-a-gMSA-for-use-with-SharpHound-Enterprise)

# Create the gMSA account

- Run the following to validate whether the domain has a KDS Root Key configured:

```
Get-KdsRootKey
```

- If there is no result returned, then the Get-KdsRootKey has not been configured in the domain

![](Pasted%20image%2020250321113728.png)

- This documentation will be using the OU examples that are documented in the SpecterOps documentation on creating the gMSA account. You will need to modify the parameters to work in your environment.
	- The **Tier0** OU
	- The two sub-OU's under the **Tier0** OU
		- **Tier0 / Groups OU**
		- **Tier0 / ServiceAccounts** OU

![](Pasted%20image%2020250321114832.png)

- The SpecterOps documentation shows the creation of the KDS Root Key using the command shown below:

```
Add-KdsRootKey -EffectiveImmediately
```

- That command is still recommended for a production environment, but the command below is being used to setup my test environment. Making the key available 10 hours ago effectively makes it available for use immediately. This prevents needing to wait 10 hours for the KDS Root Key to replicate throughout the environment (this is a lab and there is only one Domain Controller).

```
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))
```

![](Pasted%20image%2020250321115328.png)


- On a Domain Controller, run the following commands to create the **gMSA password read group**
	- Modifications will be needed to make it relevant for your environment

```
$gmsaName = "t0_gMSA_SHS" # Name of the gMSA
$pwdReadOUDN = "OU=Groups,OU=Tier0,DC=2022TESTING,DC=local" 
New-ADGroup `
-Name "$($gmsaName)_pwdRead" `
-GroupScope Global `
-GroupCategory Security `
-Path $pwdReadOUDN `
-Description "This group grants the rights to retrieve the password of the BloodHound Enterprise data collector (SharpHound Enterprise) gMSA '$gmsaName'." `
-PassThru
```


![](Pasted%20image%2020250321115636.png)

- The **gMSA password read group** has successfully been created

![](Pasted%20image%2020250321115717.png)

- On a Domain Controller, run the following PowerShell commands to add the server that will be performing the Sharphound collections to the **gMSA password read group**
	- Modifications will be needed to make it relevant for your environment


```
$gmsaName = "t0_gMSA_SHS" # Name of the gMSA
$shServerDN = "CN=SERVER2022,OU=Servers,DC=2022TESTING,DC=local" # Distinguished Name of the SharpHound Enterprise server

Add-ADGroupMember `
-Identity "$($gmsaName)_pwdRead" `
-Members $shServerDN `
-PassThru
```

![](Pasted%20image%2020250321120134.png)

- The PowerShell commands shown have all been executed on a Domain Controller running **Windows Server Core** so there is no GUI to view the OU's and groups, etc.

![](Pasted%20image%2020250321120206.png)

- When viewing the changes on a Windows Server with the GUI enabled you can see the OU's and the **t0_gMSA_SHS_pwdRead** group that has been created

![](Pasted%20image%2020250321120410.png)

- If we look at the members of the **gMSA password read group** group we see that the **SERVER2022** server has been added

![](Pasted%20image%2020250321120533.png)

- Now we are going to create the gMSA and allow the password read group to retrieve it's password
	- Use the following following PowerShell commands as a template to create the gMSA and set the retrieve right using PowerShell (run on a Domain Controller):
		- Modifications will be needed to make it relevant for your environment

```
$gmsaName = "t0_gMSA_SHS" # Name of the gMSA
$gmsaOUDN = "OU=ServiceAccounts,OU=Tier0,DC=2022TESTING,DC=local" # Distinguished Name of OU to create the gMSA in

New-ADServiceAccount -Name $gmsaName `
-Description "SharpHound Enterprise service account for BloodHound Enterprise" `
-DNSHostName "$($gmsaName).$((Get-ADDomain).DNSRoot)" `
-ManagedPasswordIntervalInDays 32 `
-PrincipalsAllowedToRetrieveManagedPassword "$($gmsaName)_pwdRead" `
-Enabled $True `
-AccountNotDelegated $True `
-KerberosEncryptionType AES128,AES256 `
-Path $gmsaOUDN `
-PassThru
```


![](Pasted%20image%2020250321120820.png)


![](Pasted%20image%2020250321120850.png)

- Now reboot the Sharphound server so that the server's membership of the `pwdRead` group takes effect.

- Test the gMSA server to make sure that the gMSA is working (this must be performed on the gMSA server)
	- Modifications will be needed to make it relevant for your environment

```
$gmsaName = "t0_gMSA_SHS" # Name of the gMSA  
  
Test-ADServiceAccount -Identity $gmsaName
```

- If the response is **True** then the gMSA account is working

![](Pasted%20image%2020250321121257.png)

# Configure the User Rights and permissions needed by the gMSA Account


## Configure the User Rights needed by the gMSA account

- Since Sharphound will be launched via a PowerShell script instead of running as a service we will need to grant the gMSA account the **Log on as a batch job** User Right instead of the **Log on as Service** User Right documented in the SpecterOps documentation. This can be done via the Local Security Policy or Group Policy.
- The screenshot below shows the gMSA account configured with the **Log on as a batch job** User Right via the Local Security Policy.

![](Pasted%20image%2020250321121808.png)

## Configure the User Permissions needed by the gMSA account

- Add the gMSA account to the Domain Admins group
	- Be sure to note that the account has a **$** at the end of it and it needs to be entered in this format when adding the account to the Domain Admins group

![](Pasted%20image%2020250321122304.png)

![](Pasted%20image%2020250321122339.png)


# Creation of the PowerShell script and scheduled task

## Creation of the PowerShell script to be run as a scheduled task

- Now the PowerShell script to run Sharphound via a scheduled task needs to be created
- The PowerShell script itself is very simple. The script contains:
	- The full path to the Shaarphound executable
	- The command line arguments that you want to pass to Sharphound
	- **Optional:** Redirection of the output to a log file (this was very helpful in getting this setup running initially and may be helpful for troubleshooting in the future)

```
C:\Users\administrator.2022TESTING\Downloads\SharpHound-v2.6.1\SharpHound.exe -c all --outputdirectory C:\Support\SharpHound --zipfilename BloodHound_gmsa.zip > C:\Support\sharphound-gmsa.log
```

![](Pasted%20image%2020250321123046.png)

## Creation of the Scheduled Task

- Open the Scheduled Tasks MSC on the Sharphound server and create a scheduled task. In the example shown below the scheduled task name is **Sharphound**

![](Pasted%20image%2020250321123246.png)

- Configure the scheduled task to run on the desired schedule
- Configure the scheduled task with the following **Action**
	- **Program / script:** `powershell.exe`
	- **Arguments:** `-ExecutionPolicy ByPass -File C:\Support\SharpHound.ps1`

![](Pasted%20image%2020250321123658.png)

- Click the **OK** button on the **Edit Action**
- Be sure to configure the scheduled task to run as a standard user account at this time
- Click the **OK** button on the scheduled task to complete the creation
- Provide the password for the standard user account when prompted and click the **OK** button

![](Pasted%20image%2020250321123944.png)


- You should now have a scheduled task to run Sharphound created. We now need to modify the scheduled task to run as the gMSA account

## Modify the scheduled task to run as the gMSA account

- At this point you should see scheduled task setup to run using the user account you previously configured

![](Pasted%20image%2020250321124316.png)

- Running the following command will modify the scheduled task to run as the gMSA account

```
schtasks /change /TN \SharpHound /RU 2022TESTING\t0_gMSA_SHS$ /RP
```

- You should see a message stating that command was successful

![](Pasted%20image%2020250321124510.png)

- If you open the scheduled task you should see that the account that it will run as is the gMSA account

![](Pasted%20image%2020250321124651.png)

# Testing the Sharphound scheduled task running as the gMSA account

- Right click on the scheduled task and select **Run**

![](Pasted%20image%2020250321124949.png)

- The scheduled task should now be running

![](Pasted%20image%2020250321125039.png)

- When completed you should have a zip file with the results

![](Pasted%20image%2020250321125129.png)

![](Pasted%20image%2020250321125204.png)

