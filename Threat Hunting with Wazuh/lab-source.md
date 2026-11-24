# Threat Hunting with Wazuh - pasted solution summary

The solution was supplied inline in chat. This file records its navigation and reported findings.

## 1. Open the historical agent records

In Firefox, open the Wazuh Dashboard on Ubuntu and sign in using the lab-provided access details. Select the Windows agent, then go to **More > MITRE ATT&CK > Framework**. Set the start date to **11 February 2026** and enable **Hide techniques with no alerts**.

The notes show one disconnected Windows agent with retained security alerts.

## 2. Investigate failed logons

Open **Brute Force**, compare timestamps, and expand the latest failed logon. Record the source IP, target IP, and account.

The notes report repeated failures from `3.110.33.3` against `192.168.1.120` for `Administrator`.

## 3. Investigate the successful network logon

Open **Pass the Hash**, expand the first successful remote logon, and compare the account and address with the failures. Record the logon type.

The notes report the same source IP and account and logon type `3`, which indicates a network logon. These details support correlation. They do not establish hash reuse.

## 4. Inspect SMB share activity

Open **SMB/Windows Admin Shares** and expand the latest event. Compare its time, source, and account with the network logon. Inspect the share field.

The notes associate `3.110.33.3` and `Administrator` with SMB share activity. The exact share name and any data transfer are unverified.

## 5. Investigate account creation

Open **Account Manipulation** and expand the latest account-creation event. Record the subject and new account, then compare the timestamp with earlier activity.

The notes report that `Administrator` created `evil-user`. This is a possible persistence indicator. The notes provide no later use or group membership evidence.
