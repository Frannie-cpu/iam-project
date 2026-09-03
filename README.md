# Identity & Access Management Lifecycle Lab — Microsoft Entra ID

**A hands-on IAM project simulating the full employee identity lifecycle — provisioning, authentication, governance, and deprovisioning — for a fictional company, NorthWind Dynamics, using Microsoft Entra ID.**

---

## Table of Contents

- [Overview](#overview)
- [Skills Demonstrated](#skills-demonstrated)
- [Environment](#environment)
- [Scenario](#scenario)
- [1. Provisioning: New Hires](#1-provisioning-new-hires)
- [2. Group-Based Access](#2-group-based-access)
- [3. Just-In-Time Privileged Access (PIM)](#3-just-in-time-privileged-access-pim)
- [4. Movers: Mid-Lifecycle Role Change](#4-movers-mid-lifecycle-role-change)
- [5. Leavers: Offboarding & the Cost of Deprovisioning Delay](#5-leavers-offboarding--the-cost-of-deprovisioning-delay)
- [6. Conditional Access Policy](#6-conditional-access-policy)
- [7. Authentication: SSO & the OAuth 2.0 Authorization Code Flow](#7-authentication-sso--the-oauth-20-authorization-code-flow)
- [8. Least-Privilege Admin Roles](#8-least-privilege-admin-roles)
- [9. Multi-Factor Authentication](#9-multi-factor-authentication)
- [10. Access Reviews](#10-access-reviews)
- [11. Conditional Access Hardening](#11-conditional-access-hardening)
- [12. Over-Permissioned Service Account](#12-over-permissioned-service-account)
- [Key Concepts Reference](#key-concepts-reference)
- [Takeaways](#takeaways)

---

## Overview

Real IAM work is a **lifecycle**: people join, their roles change, they leave — and every one of those transitions is an opportunity for either good security hygiene or an audit finding. This project walks through that full lifecycle inside Microsoft Entra ID, including the two scenarios that show up most often in real security audits: **permission creep from role changes**, and **deprovisioning delay after termination**.

## Skills Demonstrated

- User provisioning and attribute management in Microsoft Entra ID
- Group-based access management and dynamic group rules
- OAuth 2.0 / OpenID Connect authentication flow (Authorization Code flow)
- Role-Based Access Control (RBAC) and least-privilege role assignment
- Privileged Identity Management (PIM) — just-in-time admin access
- Conditional Access policy design, including break-glass account handling
- Access reviews and periodic governance
- Joiner-Mover-Leaver (JML) lifecycle management
- Service account / non-human identity risk remediation
- Security audit thinking — identifying and documenting real-world findings

## Environment

- Microsoft 365 Developer tenant (free — [developer.microsoft.com/microsoft-365/dev-program](https://developer.microsoft.com/microsoft-365/dev-program))
- Microsoft Entra admin center
- jwt.ms (Microsoft's token decoding tool) for inspecting OAuth tokens

## Scenario

NorthWind Dynamics is onboarding four new employees this week. As the IAM analyst, I'm responsible for provisioning their access correctly from day one, then carrying that responsibility through role changes, offboarding, and cleanup of risky non-human accounts.

| Employee       | Job Title    | License              | Security Group       | Admin Role                    |
|----------------|--------------|----------------------|----------------------|-------------------------------|
| Abigail Okafor | CEO          | Microsoft 365 E5     | Executives           | None                          |
| Abraham Kofi   | Receptionist | Office 365 E1        | Receptionist Team    | None                          |
| Abel Jaanus    | UT Support   | Microsoft 365 E5     | User Technology      | Helpdesk/User Admin (via PIM) |
| Abner Taavi    | HR Manager   | Microsoft 365 E5     | HR Group             | None                          |

**Design principle:**  RBAC is applied such that admin rights match actual job need — not seniority. The HR Manager — despite being the most senior person on the Human Resource (HR) team — gets **no** admin role. Seniority is not the same as system control. Only UT Support gets the Help Desk/User Admin role and only temporarily, via PIM.

An Admin role is admin power, like resetting other people's passwords. 

The principle of least privilege is simple here: **Give people the least they need.** Most employees need none at all.

---

## 1. Provisioning: New Hires

**STEPS**

- Created four user accounts in the Entra admin center with consistent naming, correct job titles/departments, and appropriate license assignment (E5 for full toolset roles, E1 for the receptionist role where a lighter license was sufficient).

> Users list in Entra admin center showing all four accounts.
> 
>![Users List](screenshots/1a-users-list-on-entraid.png)
>
> Users list in Microsoft 365 admin center showing all four accounts, licensed and active.
>
> ![Users List](screenshots/replacement.png)

---

## 2. Group-Based Access

Rather than assigning access per-user, I used **two different group types** in Entra ID, chosen deliberately based on what each was actually for:

- **Security Group** — used purely for access control: license assignment, and (for UT) admin role eligibility.
- **Microsoft 365 Group** — for a shared collaboration space: a shared inbox, shared calendar, and a private Teams space for the team to work from.

**Security Group**

**STEPS**

I created four security groups (`Executives`, `Receptionist Team`, `Human Resource Team`, `User Technology Team`) and added each relevant employee as members, to the group matching their function.

For example: - I created a **security group** for the `Human Resource Team` with only User role (no Admin Center Access role).
- Assigned the license (Office 365 E1) to the team because the embedded applications are required for members of the team to carry out their duties.
- Added **Abner Taavi** (HR Manager) to the`Human Resource Team` 

See screenshot showing how the security grouping appears for **Human Resource Team**.

> ![Users List](screenshots/security%20group%202.png)
> ![Users List](screenshots/security%20group%204.png)
> ![Users List](screenshots/security%20group%203.png)
> 

**STEPS**

I created a **security group** for the `User Technology Team` 
- Assigned the license (Microsoft 365 E5) to the team because the embedded applications are required for members of the team to carry out their duties.
- Assigned the **User** role to the `User Technology Team`  with no admin center access role. Members of the team like UT Support who needs a role to assist users with activities like password change will have to request for admin center access. See [3. Just-In-Time Privileged Access (PIM)](#3-just-in-time-privileged-access-pim)
  
- Added **Abel Jaanus** (UT Support) as a member to the `User Technology Team`

See screenshot showing how the security grouping appears for **User Technology Team**.


> ![Users List](screenshots/usertechteam2.png)
>
> ![Users List](screenshots/ut-no%20admin%20role.png) 
>
> ![Users List](screenshots/usertechteam1.png)



**Microsoft 365 Group**

**STEPS**

- I created a Microsoft 365 group for **Human Resource Team**

- I deliberately used both security and Microsoft 365 group types for **Human Resource Team** because the team had two distinct needs that don't overlap: **license/access governance** (handled by the Security Group) and **team collaboration** (handled by the Microsoft 365 Group). 


Note: The Microsoft 365 group can be used for all teams if required — for team shared collaborations, a shared inbox, shared calendar, and a private Teams space for the team to work from.

See screenshot showing how it looks on the platform. 

>![Users List](screenshots/human%20resource%20team%201.png)
>
>>![Users List](screenshots/human%20resource%20team%202.png)


## 3. Just In-Time Privileged Access-PIM
 
Abel Jaanus (UT support) needs elevated access, to enable them perform admin duties like password change etc. They can temporarily apply for this role within a specified period, after which access switches off by itself. This is called just-in-time access.

**Why this matters:** if a hacker steals Abel's password while the role is switched off, they get only an ordinary account, not admin power.

**STEPS**

- I added a new assignment and added the **Helpdesk Administrator** role.**Global Administrator** role will be excessive for UT Support, so **helpdesk administrator** is suitable. 

> ![Users List](screenshots/addrole.png)

-  I added Abel Jaanus as a member to the role and set assignment type to `Eligible` and not `Active` . This way Abel is only eligible and needs to authenticate and activate, the assigned role.
 
> ![Users List](screenshots/addmember.png)

-  I setup the assignment maximum duration to 4 hours, to require multi-factor authentication on activation, require justification, and
require approval to activate.

> ![Users List](screenshots/rolesettings.png)
>
> Abel is now `Eligible`  for the assigned role but needs to activate the role on their end before they can use it.
> 
> ![Users List](screenshots/eligibleassignment.png)
>
> 
> ![Users List](screenshots/noactiveassignment.png)



## 4. Movers: Mid-Lifecycle Role Change

A new team group `IAM Team` was introduced here. 
| What Happened | Who | What I Did |
|---|---|---|
| Promotion (a Mover) | Abraham Kofi, from Receptionist to IAM Specialist | I updated his job details, added him to the IAM Specialist group, and removed him from the old Receptionist group. |


**STEPS**

- I updated Abraham Kofi's job details, so the system knows he moved from the `Receptionist Team` to a new team `IAM Team`
> ![Users List](screenshots/updatedetails.png)

- I gave him his new access by adding him to the `IAM Team` group

> ![Users List](screenshots/addabraham2.png)

- I took away the old `Receptionist Team` access , he no longer needs. Abraham is no longer a Receptionist, so he should not keep his old access. If the access is not removed, he keeps collecting access for years, which is called **privilege creep**. This is how an ordinary account slowly turns into a very dangerous one.

> ![Users List](screenshots/removeabraham.png)

## 5. Leavers: Offboarding & the Cost of Deprovisioning Delay

| What Happened | Who | What I Did |
|---|---|---|
| Resignation (a Leaver) | Abner Taavi, HR Manager | I blocked his sign-in, signed him out everywhere, removed his groups, saved his work, and took back his licence. |

## 6. Conditional-Access-Policy
conditional access
