
#Bloodhound
#Sharphound
#Discovery
#AttackPaths
#Docker 

The purpose of this page is to document the steps needed to setup Sharphound (for the Community Edition) to run as a scheduled task using a gMSA (Group Managed Service Account)

# Resources

[Create a gMSA for use with SharpHound Enterprise](https://support.bloodhoundenterprise.io/hc/en-us/articles/9295298941723-Create-a-gMSA-for-use-with-SharpHound-Enterprise)

# Create the gMSA account

- Run the following to validate whether the domain has a KDS Root Key configured:

```
Get-KdsRootKey
```

- If there is no result returned, then the Get-KdsRootKey has not been configured in the domain

![Pasted image 20250321113728](https://github.com/user-attachments/assets/503d5c11-baee-4818-b529-5aef0ab26dc1)


- This documentation will be using the OU examples that are documented in the SpecterOps documentation on creating the gMSA account. You will need to modify the parameters to work in your environment.
	- The **Tier0** OU
	- The two sub-OU's under the **Tier0** OU
		- **Tier0 / Groups OU**
		- **Tier0 / ServiceAccounts** OU

![Pasted image 20250321114832](https://github.com/user-attachments/assets/5a103c0e-2d7b-4554-a9ef-31140c651659)


- The SpecterOps documentation shows the creation of the KDS Root Key using the command shown below:

```
Add-KdsRootKey -EffectiveImmediately
```

- That command is still recommended for a production environment, but the command below is being used to setup my test environment. Making the key available 10 hours ago effectively makes it available for use immediately. This prevents needing to wait 10 hours for the KDS Root Key to replicate throughout the environment (this is a lab and there is only one Domain Controller).

```
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))
```

![Pasted image 20250321115328](https://github.com/user-attachments/assets/fafe870e-2e6f-46c3-8ee4-4b9c70233e29)


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

![Pasted image 20250321115636](https://github.com/user-attachments/assets/3a5a56fb-3346-42e8-ac81-36cec32dca0a)



- The **gMSA password read group** has successfully been created

![Pasted image 20250321115717](https://github.com/user-attachments/assets/f43b89bf-003f-482b-9f3f-120090ad49dc)


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

![Pasted image 20250321120134](https://github.com/user-attachments/assets/e26665cb-5244-40d0-bb4a-0ece721081fc)


- The PowerShell commands shown have all been executed on a Domain Controller running **Windows Server Core** so there is no GUI to view the OU's and groups, etc.

![Pasted image 20250321120206](https://github.com/user-attachments/assets/247244ce-695d-40a3-8d89-5089ff60c2af)


- When viewing the changes on a Windows Server with the GUI enabled you can see the OU's and the **t0_gMSA_SHS_pwdRead** group that has been created


![Pasted image 20250321120410](https://github.com/user-attachments/assets/72497647-a33c-400f-b5e1-1094d608358d)


- If we look at the members of the **gMSA password read group** group we see that the **SERVER2022** server has been added

![Pasted image 20250321120533](https://github.com/user-attachments/assets/f87230ed-dc02-418f-9beb-4e233f62e45e)


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

![Pasted image 20250321120820](https://github.com/user-attachments/assets/33662789-3d9c-4377-a090-450c91adf855)


![Pasted image 20250321120850](https://github.com/user-attachments/assets/4e8bbad1-a058-48c0-870b-a580b749cf4e)


- Now reboot the Sharphound server so that the server's membership of the `pwdRead` group takes effect.

- Test the gMSA server to make sure that the gMSA is working (this must be performed on the gMSA server)
	- Modifications will be needed to make it relevant for your environment

```
$gmsaName = "t0_gMSA_SHS" # Name of the gMSA  
  
Test-ADServiceAccount -Identity $gmsaName
```

- If the response is **True** then the gMSA account is working

![Pasted image 20250321121257](https://github.com/user-attachments/assets/878abf53-575b-4d5b-a0f7-3e652da869f2)


# Configure the User Rights and permissions needed by the gMSA Account


## Configure the User Rights needed by the gMSA account

- Since Sharphound will be launched via a PowerShell script instead of running as a service we will need to grant the gMSA account the **Log on as a batch job** User Right instead of the **Log on as Service** User Right documented in the SpecterOps documentation. This can be done via the Local Security Policy or Group Policy.
- The screenshot below shows the gMSA account configured with the **Log on as a batch job** User Right via the Local Security Policy.

![Pasted image 20250321121808](https://github.com/user-attachments/assets/ebd1d4eb-a8e7-40f2-a212-77051d1caa18)


## Configure the User Permissions needed by the gMSA account

- Add the gMSA account to the Domain Admins group
	- Be sure to note that the account has a **$** at the end of it and it needs to be entered in this format when adding the account to the Domain Admins group

![Pasted image 20250321122304](https://github.com/user-attachments/assets/498bd5ae-d535-4fab-8855-7d77bc70569a)


![Pasted image 20250321122339](https://github.com/user-attachments/assets/2927ad92-d077-4aed-8298-ea9112fc8333)


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


![Pasted image 20250321123046](https://github.com/user-attachments/assets/34784ec7-d839-444c-abb4-b08e31dd6182)


## Creation of the Scheduled Task

- Open the Scheduled Tasks MSC on the Sharphound server and create a scheduled task. In the example shown below the scheduled task name is **Sharphound**


![Pasted image 20250321123246](https://github.com/user-attachments/assets/4264f48f-7bd7-442d-af6f-f0d3f6e07c0d)


- Configure the scheduled task to run on the desired schedule
- Configure the scheduled task with the following **Action**
	- **Program / script:** `powershell.exe`
	- **Arguments:** `-ExecutionPolicy ByPass -File C:\Support\SharpHound.ps1`


![Pasted image 20250321123658](https://github.com/user-attachments/assets/e29ee013-2f35-4758-b1de-baeef5af462a)


- Click the **OK** button on the **Edit Action**
- Be sure to configure the scheduled task to run as a standard user account at this time
- Click the **OK** button on the scheduled task to complete the creation
- Provide the password for the standard user account when prompted and click the **OK** button

![Pasted image 20250321123944](https://github.com/user-attachments/assets/a765b9b5-6b23-4132-a0d3-8a62c8987e97)


- You should now have a scheduled task to run Sharphound created. We now need to modify the scheduled task to run as the gMSA account

## Modify the scheduled task to run as the gMSA account

- At this point you should see scheduled task setup to run using the user account you previously configured

![Pasted image 20250321124316](https://github.com/user-attachments/assets/f38fa808-6917-4d25-b144-54884be9fcae)


- Running the following command will modify the scheduled task to run as the gMSA account

```
schtasks /change /TN \SharpHound /RU 2022TESTING\t0_gMSA_SHS$ /RP
```

- You should see a message stating that command was successful


![Pasted image 20250321124510](https://github.com/user-attachments/assets/a635c92d-9fe0-42b8-8906-d2bb152a9485)


- If you open the scheduled task you should see that the account that it will run as is the gMSA account

![Pasted image 20250321124651](https://github.com/user-attachments/assets/544d5baa-1fc4-4614-b1da-ce172dd59106)


# Testing the Sharphound scheduled task running as the gMSA account

- Right click on the scheduled task and select **Run**

![Pasted image 20250321124949](https://github.com/user-attachments/assets/72bf2519-818b-4f5b-817d-b731f80db634)


- The scheduled task should now be running

![Pasted image 20250321125039](https://github.com/user-attachments/assets/96df1d97-d1df-42ff-8a37-b94542c952b6)


- When completed you should have a zip file with the results

![Pasted image 20250321125129](https://github.com/user-attachments/assets/05487c09-ae7a-4117-bc64-d07919aacc26)


![Pasted image 20250321125204](https://github.com/user-attachments/assets/2508f3e5-9eb2-426a-b8c4-7c1144462876)



