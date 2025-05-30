---
description: How to grant/deny access to certain runbooks.
---

# Runbook Permissions

{% embed url="https://www.youtube.com/watch?v=bLi_k_Yzhyw" %}

## Scope

This addresses how to grant/deny access to certain runbooks in an Azure tenant. If you are looking for answers regarding which MS Graph API permissions are needed to run a certain action as a runbook, please have a look at our [requirements](../azure-ad-roles-and-permissions.md).

## Overview

"Runbook Permissions" define the visibility of runbooks for certain users. Certain runbooks can also be blocked/hidden globally.

Like the [Runbook Customizations](runbook-customization.md), defining these permissions is done by giving a JSON-formatted configuration as an RealmJoin admin in RealmJoin's web portal at [https://portal.realmjoin.com/settings/runbooks-permissions](https://portal.realmjoin.com/settings/runbooks-permissions) .

### About this guide

We will give a short description of the syntax and then build a full example piece by piece. Feel free to directly go to the full sample and start from [there](runbook-permissions.md#targetentitygroups).

## Configuration Syntax

### Runbook names

Runbooks are referenced by their names as seen in the Azure Automation Account, e.g. `rjgit-group_general_remove-group`.

Wildcards ('\*') can be used to match multiple runbooks. Multiple wildcards can be used in the same string, e.g. `rjgit-*_security_*`. This would all of the following examples:

* `rjgit-org_security_list-inactive-users`
* `rjgit-device_security_enable-or-disable-device`

The prefix `rjgit-` denotes runbooks that are imported from our piblic GitHub repository. Customer-specific runbooks have no prefix, e.g. `user_userinfo_custom-runbook`

### Entra ID Groups

Entra ID groups will be referenced using their Object ID, like `91688d11-9a34-42cd-8d1e-ce617d6c1234`. Currently, only security groups can be used.

## JSON Structure and Example

We will build a full configuration example piece by piece.

A JSON configuration consists of multiple sections, but all sections are optional and can be omitted.

It is allowed to add comments using the "//" prefix.

### EnabledRunbookPatterns

This section contains a list of runbooks allowed to be used. If this section is omitted, all runbooks are enabled/allowed by default.

If you define this section, then only runbooks mentioned in this section will be usable by any role / support and admin.

#### Example

*   Allow only certain, individual runbooks by giving their full name

    `rjgit-group_general_remove-group`
*   Allow all device related runbooks from our shared repository

    `rjgit-device_*`
*   Allow all shared user runbooks

    `rjgit-user_*`
*   Allow all customer specific (local), user related runbooks

    `user_*`

This implicitely leaves out many group- and all org- based runbooks. Be aware.

```
{
  "EnabledRunbookPatterns": [
    "rjgit-group_general_remove-group",
    "rjgit-device_*",
    "rjgit-user_*",
    "user_*"
  ]
}
```

### DisabledRunbookPatterns

A list of runbook that are globally disabled / forbidden. If this section is omitted or empty, all enabled runbooks (given via [EnabledRunbookPatterns](runbook-permissions.md#enabledrunbookpatterns)) are usable.

Entries in this section take priority over entries in [EnabledRunbookPatterns ](runbook-permissions.md#enabledrunbookpatterns)- the runbooks will be hidden/not be usable for anyone in this tenant.

#### Example

We will reuse the `EnabledRunbookPatterns` section from before.

* Disable all shared (`rjgit-`) runbooks in the `security` category.

```
{
  "EnabledRunbookPatterns": [
    "rjgit-group_general_remove-group",
    "rjgit-device_*",
    "rjgit-user_*",
    "user_*"
  ],
  "DisabledRunbookPatterns": [
    "rjgit-*_security_*"
  ]
}
```

### Roles

In this section you can assign a list of runbooks to an Entra ID group. This allows to define multiple support/operator roles in your tenant.

If this section is omitted, all RealmJoin support and administrators have access to all runbooks given in the previous sections.

{% hint style="warning" %}
If enabled, all users not belonging to a role will not see any runbooks.
{% endhint %}

#### Example

Continuing with what we have, let us create a device-support role `DeviceAdmin` and a user support role `UserAdmin`.

We will apply those roles to multiple Entra ID groups and for each role give a list of allowed runbooks. Be aware - this will restrict the user support role to only a small set of runbooks.

Let us add comments ("//") next to the group's object id that help the reader by giving the Entra ID group names.

```json
{
  "EnabledRunbookPatterns": [
    "rjgit-group_general_remove-group",
    "rjgit-device_*",
    "rjgit-user_*",
    "user_*"
  ],
  "DisabledRunbookPatterns": [
    "rjgit-*_security_*"
  ],
  "Roles": {
    "DeviceAdmin": {
      "Groups": [
        "9cbfc0af-c217-41e9-b790-3043788f1234", // 1st Device Support AAD Group
        "5555c0af-c217-41e9-b790-3043788f1234"  // 2nd Device Support AAD Group - other team
      ],
      "AllowedRunbookPatterns": [
        "rjgit-device_*"
      ]
    },
    "UserAdmin": {
      "Groups": [
        "1234c0af-c217-41e9-b790-3043788f1234" // User Support AAD Group
      ],
      "AllowedRunbookPatterns": [
        "rjgit-user_general_assign-or-unassign-license",
        "rjgit-user_mail_*",
        "user_*"
      ]
    }
  }
}
```

Now the `UserAdmin` role can:

* assign licenses to all users in your tenant
* modify email-addresses of all users in your tenant

The `DeviceAdmin` role can

* wipe any device in your tenant

### TargetEntityGroups

Perhaps you have some crucial VIP users. It should not be possible for just any support staff to erase a VIP's device or modify a VIP's email address. We can use "targeting" to restrict roles on critical users to dedicated teams.

"Devices" will be targeted by their primary/assigned user but not the device object in Entra ID. This allows to stick to a purely user-based group model.

We assume, Entra ID groups exist that contain critical VIP users. Using this section, we can carefully scope some more critical roles and runbooks for these specific Entra ID groups (targets).

Obviously, if you omit this section, all users/groups/devices in your tenant are treated as equals.

If you define TargetEntityGroups it should not have any impact on any other group not mentioned in the section.

#### Full Example

Assume group `0000c0af-c217-41e9-b790-3043788f0000` is our group of VIP users.

We introduce a new Entra ID group `4444c0af-c217-41e9-b790-3043788f4444` containing supporting staff that have been approved to administrate VIP users. These supporting staff should also have all other basic support permissions, so we will add them to the existing roles.

"Restricting" a role will not grant new roles to a supporter.

```json
{
  "EnabledRunbookPatterns": [
    "rjgit-group_general_remove-group",
    "rjgit-device_*",
    "rjgit-user_*",
    "user_*"
  ],
  "DisabledRunbookPatterns": [
    "rjgit-*_security_*"
  ],
  "Roles": {
    "DeviceAdmin": {
      "Groups": [
        "9cbfc0af-c217-41e9-b790-3043788f1234", // 1st Device Support AAD Group
        "5555c0af-c217-41e9-b790-3043788f1234", // 2nd Device Support AAD Group - other team
        "4444c0af-c217-41e9-b790-3043788f4444"  // VIP Support Crew
      ],
      "AllowedRunbookPatterns": [
        "rjgit-device_*"
      ]
    },
    "UserAdmin": {
      "Groups": [
        "1234c0af-c217-41e9-b790-3043788f1234", // User Support AAD Group
        "4444c0af-c217-41e9-b790-3043788f4444"  // VIP Support Crew
      ],
      "AllowedRunbookPatterns": [
        "rjgit-user_general_assign-or-unassign-license",
        "rjgit-user_mail_*",
        "user_*"
      ]
    }
  },
  "TargetEntityGroups": {
    "0000c0af-c217-41e9-b790-3043788f0000": {  // VIP Users - Treat with care!
      "RestrictRoles": {
        "UserAdmin": [
          "4444c0af-c217-41e9-b790-3043788f4444" // VIP Support
        ],
        "DeviceAdmin": [
          "4444c0af-c217-41e9-b790-3043788f4444" // VIP Support
        ]
      }
    }
  }
}
```

#### **Example: Restricting US Support Staff to Manage Only US Users**

In this scenario, we have US-based Support Staff who should only manage Users located in the US. To enforce this restriction:

* Create a permission rule that explicitly **denies** US Supporters the ability to execute Runbooks on **all Users**.
* Add an exception rule that specifically **allows** execution of Runbooks only for **US Users**.

This ensures US Supporters have permissions limited strictly to their intended target audience (US Users) and prevents accidental interactions with users outside this scope.

**Implementation**

1. A Runbook Runners Entra group must be assigned in Realm Join Portal
   1. Settings > Permissions > Runbook Runner Permissions
   2. US Supporters Entra group must be member of the Runbook Runners Group to allow general Runbook operation in the RealmJoin Portal.
2. Adding a new Role under Settings > Runbook Permissions
   1. In the Roles section add USSupporters Role with their Entra group (group object ID)
   2. Add AllowedRunbookPatterns for the USSupporters
3. Modify the TargetEntityGroups
   1. All-Users group must Restrict the Role USSupporters with an empty value (no Entra group object ID is added here). This is an implicit denial!
   2. US Users group must Restrict the Role USSupporters to the US Supporters Entra group object ID



<figure><img src="../../.gitbook/assets/image (24).png" alt=""><figcaption><p>Restricting US Support Staff to manage only US Users</p></figcaption></figure>

Below the complete example for this scenario:

```json
{
  // Portal Permission:
  // Runbook Runner Role: US Supporters

  // Group Memberships:
  // 3e1e7540-7f0c-483c-b9bf-500342e2467c: All-Users
  // c603278c-cc36-4661-bb7a-eecb7ab079f9: US Supporters
  // 0f76d01e-cc6b-4553-bf1d-e4ccedd9c824: US Users

  "EnabledRunbookPatterns": [ // General enablement of mail and security Runbooks
    "rjgit-*_mail_*",
    "rjgit-*security*"
  ],
  "DisabledRunbookPatterns": [
    "*password*"
  ],

  "Roles": {
    "USSupporters": {
      "Groups": [
        "c603278c-cc36-4661-bb7a-eecb7ab079f9" // US Supporters
      ],
      "AllowedRunbookPatterns": [ // allowed Runbooks for this Role - US Supporters
        "rjgit-user_*",
        "rjgit-device_*",
        "rjgit-group_*"
      ]
    }
  },
  
  "TargetEntityGroups": {
    "3e1e7540-7f0c-483c-b9bf-500342e2467c": { // All-Users
      "RestrictRoles": {
        "USSupporters": [
          // generally, do not allow Runbooks for any US Supporters
        ]
      }
    },
    "0f76d01e-cc6b-4553-bf1d-e4ccedd9c824": { // US Users
      "RestrictRoles": {
        "USSupporters": [
          "c603278c-cc36-4661-bb7a-eecb7ab079f9" // US Supporters
        ]
      }
    }
  }
}
```

### SchedulingEnabledRunbookPatterns

This section contains a list of runbooks that will be flagged as "schedulable". RealmJoin Port will allow to assign / manage schedules for these runbooks. See [scheduling.md](scheduling.md "mention").

The following example describes the default behaviour if SchedulingEnabledRunbookPatterns are not defined:

```json
{
  "SchedulingEnabledRunbookPatterns": [
    "*_scheduled"
  ]
}
```

### SchedulingDisabledRunbookPatterns

This section contains a list of runbooks that will be blacklisted from being flagged as "schedulable". RealmJoin Port will not allow to assign / manage schedules for these runbooks. See [scheduling.md](scheduling.md "mention").

A runbook present in both SchedulingEnabledRunbookPatterns and SchedulingDisabledRunbookPatterns will **not** be schedulable.&#x20;

By default no runbooks are blacklisted. The following example just demonstrates the syntax:

```json
{
  "SchedulingDisabledRunbookPatterns": [
    "rjgit-user_*"
  ]
}
```
