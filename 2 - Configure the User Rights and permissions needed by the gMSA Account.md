---
share: "true"
---
  
	  
#Bloodhound  
#Sharphound  
#Discovery  
#AttackPaths  
#Docker   
  
  
# Configure the User Rights needed by the gMSA account  
  
- Since Sharphound will be launched via a PowerShell script instead of running as a service we will need to grant the gMSA account the **Log on as a batch job** User Right instead of the **Log on as Service** User Right documented in the SpecterOps documentation. This can be done via the Local Security Policy or Group Policy.  
- The screenshot below shows the gMSA account configured with the **Log on as a batch job** User Right via the Local Security Policy.  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321121808.png)  
  
# Configure the User Permissions needed by the gMSA account  
  
- The Active Directory details collected by Sharphound depend on the permissions that the user running Sharphound has within the Domain  
	- A regular, non-privileged user can run Shaprhound and collect a significant amount of information, but some of the **Edges** can only be collected by users with elevated permissions within the Domain  
- At this time there are two main paths that could be used to give the Sharphound gMSA account the permissions needed to collect the Active Directory information needed  
	- **Method #1** - Make the Sharphound gMSA account a member of the **Local Administrators** group on all computers in the domain  
	- **Method #2** - Make the Sharphound gMSA account a member of the **Domain Admins** group  
- At this time, it is my understanding that using an account that has **Domain Admins** permissions will enable the Sharphound executable to collect all of the possible pertinent information from Active Directory for the best possible analysis by BloodHound. This may change in the future.  
  
## gMSA Account protections  
  
- Regardless of whether the permissions that are given to the Sharphound gMSA account are using **Method #1** or **Method #2**, additional protections should be considered  
	- **NOTE:** See SpecterOps documentation on [hardening the Sharphound service](https://bloodhound.specterops.io/manage-bloodhound/securing-bloodhound-and-collectors/sharphound-hardening). This documentation contains information on a number of protections that can be put in place to prevent the Sharphound gMSA account from being compromised and leveraged by an attacker.  
- In either situation the gMSA account should be a member of the Active Directory **Protected Users** group  
- [Tiering the Sharphound gMSA account](https://bloodhound.specterops.io/install-data-collector/install-sharphound/tiered-collector-strategy) access is another methodology that can be implemented  
  
## Configuring the gMSA account as a Domain Admin  
  
- Add the gMSA account to the **Domain Admins** group  
	- Be sure to note that the account has a **$** at the end of it and it needs to be entered in this format when adding the account to the Domain Admins group  
	- At this time the gMSA account should also be added to the **Protected Users** group. If you are not familiar with the Active Directory **Protected Users** groups, refer to the [Microsoft Documentation](https://learn.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group). There are a number of benefits to using this group to protect privileged accounts from being vulnerable to NTLM Relay attacks and other credential attacks.  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321122304.png)  
  
![](./PenTesting/Bloodhound%20&%20SharpHound/_resources/Pasted%20image%2020250321122339.png)  
  
