---
share: "true"
---
  
#Bloodhound  
#Sharphound  
#Discovery  
#AttackPaths  
#Docker  
  
# Resources  
  
[Create a gMSA for use with SharpHound Enterprise](https://support.bloodhoundenterprise.io/hc/en-us/articles/9295298941723-Create-a-gMSA-for-use-with-SharpHound-Enterprise)  
[Deploy gMSA account as task scheduler user account](https://stackoverflow.com/questions/62699123/deploy-gmsa-account-as-task-scheduler-user-account)  
[Using Group Managed Services Account with Scheduled Tasks](https://thesleepyadmins.com/2024/02/05/using-group-managed-services-account-with-scheduled-tasks/)  
  
  
# Creating the gMSA account  
  
- Run the following to validate whether the domain has a KDS Root Key configured:  
  
```  
Get-KdsRootKey  
```  
  
- If there is no result returned, then the KDS Root Key has not been configured in the domain  
- If there is a result returned, the the KDS Root Key has already been configured in the domain, and the step to create the KDS Root Key using the `Add-KdsRootKey` command can be skipped  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321113728.png)  
  
- This documentation will be using the OU examples that are documented in the SpecterOps documentation on creating the gMSA account. You will need to modify the parameters to work in your environment.  
	- The **Tier0** OU  
	- The two sub-OU's under the **Tier0** OU  
		- **Tier0 / Groups OU**  
		- **Tier0 / ServiceAccounts** OU  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321114832.png)  
  
- The SpecterOps documentation shows the creation of the KDS Root Key using the command shown below:  
  
```  
Add-KdsRootKey -EffectiveImmediately  
```  
  
- That command is still recommended for a production environment, but the command below is being used to setup my test environment. Making the key available 10 hours ago effectively makes it available for use immediately. This prevents needing to wait 10 hours for the KDS Root Key to replicate throughout the environment (this is a lab and there is only one Domain Controller).  
  
```  
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))  
```  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321115328.png)  
  
  
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
  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321115636.png)  
  
- The **gMSA password read group** has successfully been created  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321115717.png)  
  
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
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321120134.png)  
  
- The PowerShell commands shown have all been executed on a Domain Controller running **Windows Server Core** so there is no GUI to view the OU's and groups, etc.  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321120206.png)  
  
- When viewing the changes on a Windows Server with the GUI enabled you can see the OU's and the **t0_gMSA_SHS_pwdRead** group that has been created  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321120410.png)  
  
- If we look at the members of the **gMSA password read group** group we see that the **SERVER2022** server has been added  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321120533.png)  
  
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
  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321120820.png)  
  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321120850.png)  
  
- Now reboot the Sharphound server so that the server's membership of the `pwdRead` group takes effect.  
  
## Validating that the gMSA account is working  
  
- **NOTE:** The `Test-ADServiceAccount` PowerShell command is dependent on the PowerShell AD RSAT cmdlets be install on the system where the gMSA account will be running Sharphound  
	- SpecterOps notes about this can be found [here](https://bloodhound.specterops.io/install-data-collector/install-sharphound/create-gmsa#test-the-gmsa-optional)  
- Test the gMSA server to make sure that the gMSA is working (this must be performed on the gMSA server)  
	- Modifications will be needed to make it relevant for your environment  
  
```  
$gmsaName = "t0_gMSA_SHS" # Name of the gMSA    
    
Test-ADServiceAccount -Identity $gmsaName  
```  
  
- If the response is **True** then the gMSA account is working  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321121257.png)  
  
