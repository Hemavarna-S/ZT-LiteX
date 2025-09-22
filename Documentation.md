
# Building a Zero Trust Simulation in Azure Free Tier

---

## Diving Into Azure: My Zero Trust Journey

Finally, after exploring Azure for a couple of months, I decided to get hands-on and build a Zero Trust simulation.

I jumped in without prior hands-on experience, and it wasn’t easy! Creating users in **Entra ID (formerly Azure Active Directory)**, adding them to groups, and assigning licenses was challenging at first. But through trial and error, I learned a lot.

In this blog, I’ll share my journey of implementing Zero Trust in Azure. I made several mistakes, mixing up Application IDs, Client IDs, and misconfiguring App Services. After many errors and 9+ hours of effort, I finally got it working and learned valuable lessons.

 **The biggest takeaway?** Mistakes are not failures — they are opportunities to learn, grow, and strengthen your patience and *never give up mindset*.

---

## Understanding Zero Trust

### Traditional Security (Mall Analogy)
- Imagine the mall has one big security gate at the entrance.  
- Once you get inside, no one checks you again, you can go anywhere, into any shop.  
- This is like the old security model: once someone is *inside the network*, they’re trusted everywhere.  
- If a criminal sneaks in, they can move freely and cause damage.  

### Zero Trust Security (New Model)
- Now imagine a Zero Trust mall. Yes, you pass through security at the entrance, but that’s not enough:  
  - To enter a shop, you may need to scan your digital pass.  
  - To access the staff-only storage room, you’ll need special access approval.  
  - Security cameras monitor continuously, and your actions are verified at every step.  

Even if someone manages to enter, they can’t freely move everywhere. Access is always checked and **“never trusted, always verified.”**

---

## Why Zero Trust Matters

- In the digital world, attackers often break through one weak spot.  
- With Zero Trust, **every door, every system, every request is checked**.  
- Even employees or known users get only what they need, when they need it.  
- This minimizes risk, protects sensitive data, and limits damage if attackers sneak in.  

---

## Tech Stack (What I Used)

- **Identity & Access**: Microsoft Entra ID (no P1), MFA (per-user), security groups  
- **App Hosting**: App Service (F1 free tier) with Access Restrictions (IP allow/deny)  
- **Visibility**: Log Analytics workspace (ZT-Logs), KQL queries  
- **Detection & Posture**: Defender for Cloud (review Secure Score)  

---

## Prerequisites & Assumptions

- Azure subscription (**Free Tier is fine**)  
- Create a budget to avoid unexpected charges  
<img width="1100" height="495" alt="image" src="https://github.com/user-attachments/assets/546d5f8c-6d69-4863-be0e-0208abc74802" />
<img width="953" height="860" alt="image" src="https://github.com/user-attachments/assets/ccd67fd8-696e-4409-97c2-237e4a51e469" />



---

## Step 1: Create Identities & Groups

1. Open **Azure portal → Microsoft Entra ID → Users → + New user**  
   - Create: `user1`, `user2`, `secadmin`, `breakglass`
    <img width="1100" height="523" alt="image" src="https://github.com/user-attachments/assets/ea620d91-b149-470b-b36c-ad3bcff816cb" />
 

2. Create groups: **Entra ID → Groups → + New group**
   - **AllEmployees** (members: user1, user2)  
   - **SecurityAdmins** (member: secadmin)
     <img width="1100" height="524" alt="image" src="https://github.com/user-attachments/assets/f1c6eae4-e465-4c8b-a3d4-bf26663afc77" />
      <img width="1100" height="496" alt="image" src="https://github.com/user-attachments/assets/ec08c865-ff92-483f-8187-1e67776a9d93" />


3. Keep `breakglass` in a separate emergency account (protect carefully).  

4. Enable MFA (since no P1):  
   - Go to **Entra ID → Users → Multi-Factor Authentication (classic portal link)**  
   - Enable MFA for chosen users (e.g., user1, user2).  

💡 **Tip**: Keep `breakglass` with strong protection and store credentials in **Key Vault** or a password vault.  
<img width="1100" height="494" alt="image" src="https://github.com/user-attachments/assets/12980aa9-a46f-425d-b63b-1a0b0d5fb975" />
<img width="1100" height="549" alt="image" src="https://github.com/user-attachments/assets/97219f00-9678-4b1d-907c-7dcb1c05b4f0" />
<img width="1100" height="555" alt="image" src="https://github.com/user-attachments/assets/e7739047-65f2-475a-99e1-6d2af2e407e2" />




---

## Step 2: Register the App & Assign Access

1. Go to **Microsoft Entra ID → App registrations**   
<img width="1100" height="500" alt="image" src="https://github.com/user-attachments/assets/4f697315-67d2-4da1-b5be-bc5df45f0ffa" />

2. Click **+ New application → Create your own application**  
   - Name the app: `ZT-DemoWebApp`  
3. In the app’s Properties:  
   - Set **User assignment required? → Yes**  
   - This ensures only assigned users or groups can access the app  
4. Assign access:  
   - Click **+ Add user/group**  
   - Select the group (e.g., AllEmployees)  
   - ⚠️ In Free Tier, you may need to add individual users instead.  
<img width="1100" height="475" alt="image" src="https://github.com/user-attachments/assets/44ec5e2b-ebab-4d0a-9e0f-fa8d7ecac0cb" />

---

## Step 3: Deploy App Service & Lock Network Access

1. Create **App Service (F1 free tier)**  
<img width="1100" height="424" alt="image" src="https://github.com/user-attachments/assets/b5ef75e1-703d-497f-8c94-463502da9be0" />

2. Go to: **App Service → Networking → Access restrictions → Add rule**  

   - Allow Rule:  
     - Name: `Allow-Home-IP`  
     - Action: **Allow**  
     - Source: IPv4 (set your public IP)  
     - Priority: 100
      <img width="1100" height="494" alt="image" src="https://github.com/user-attachments/assets/81be0ab2-f7c0-4c7e-a47e-7c67e8ba32f5" />
 

   - Deny Rule:  
     - Action: **Deny**  
     - Priority: 200 (lower priority)
        <img width="1100" height="498" alt="image" src="https://github.com/user-attachments/assets/7560fe7d-7838-40fb-afdf-8e8e5a096e80" />


3. Test access:  
   - From your allowed IP → Should succeed  
   - From another network → Should be denied  

---

## Step 4: Visibility, Alerts & Hardening

1. Create **Log Analytics Workspace** → Name it `ZT-logs`  
2. Link your **App Service** and **Entra ID sign-in logs** to this workspace.  
3. Run a simple KQL query:

```kusto
SigninLogs
| where ResultType != 0
| summarize FailedAttempts = count() by UserPrincipalName, IPAddress

```
<img width="1100" height="498" alt="image" src="https://github.com/user-attachments/assets/6257bf1e-8476-4658-9d62-4289bec8c4a2" />
---
<img width="1100" height="475" alt="image" src="https://github.com/user-attachments/assets/cb732210-2182-4413-909f-a49666ed855e" />
---
<img width="1100" height="498" alt="image" src="https://github.com/user-attachments/assets/dde0fa4a-e91b-4a53-b0b8-985e8bb7e7d6" />
---
<img width="1100" height="699" alt="image" src="https://github.com/user-attachments/assets/3fb61c40-74d4-48bf-b52f-fc4a7dd300b4" />



