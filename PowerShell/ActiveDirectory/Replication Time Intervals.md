# Active Directory Replication Time Intervals

**Intrasite Replication:** 

- When a directory change is made, the source DC waits **15 seconds** before it sends the update notification to closest replication partner
- If there is more than one replication partner, the changes go out in **3 second increments** to the subsequent replication partners
- After receiving notification of the change, the partner domain controller sends a directory update request to the source domain controllers.  The source DC responds with a replication operation.  The 3 second skew prevents overloading the source DC from replication partners if there are many.
- There are of course exceptions to the **15 second** time frame where this doesn’t apply and replication occurs immediately. This is known as **urgent replication**, this immediate [replication applies to critical directory](https://www.virtualizationhowto.com/2023/04/check-server-replication-status-in-active-directory/) **updates**.  The include account lockouts and changes in the account lockout policy, the domain password policy, or the password on a domain controller account or user passwords.

**Windows 2000** (Yes it is still out there) 

- Default delay with Windows 2000 DCs for intrasite replication is **5 minutes**

**Intersite Replication** 

- By default intersite replication occurs between each site every **180 minutes or 3 hours**
- The lowest interval that intersite replication can be adjusted to is **15 minutes** 

**Group Policy replication interval** 

- Changes to Group Policy settings might not be immediately available as they have to replicate to the appropriate domain controller.
- Clients have a **90-minute refresh** period (randomized by up to approximately **30 minutes**)
- Components of GPO are stored in both AD and on the Sysvol folder of domain controllers.
- You can manually trigger a GPO refresh with the **gpupdate** command with XP and up, or with **secedit** with Windows 2000 environments

**SYSVOL & FRS/DFS** 

- SYSVOL replication is state based meaning replication happens as soon as anything changes in the SYSVOL folders  Replication pre Windows 2008 is taken care of via the File Replication Service (FRS) and then starting with Windows 2008 domain functional level you can use DFS technology to replicate SYSVOL information.

**DNS Replication Active Directory** 

- If DNS zones are AD integrated it is updated using AD replication.  Any new DNS record that is created in AD integrated zone is replicated immediately with AD intra-site replication.
- The DNS record **will not appear immediately** however even though the AD database is up to date.
- The DNS server does not query the AD database directly but every **180 seconds** it reloads the zone from the latest AD database values.
- You can check your **dwDsPollingInterval** attribute by using the command **dnscmd /info**
- **Force DNS polling** – use the command **dnscmd /zoneupdatefromds _yourzonenamehere_**
