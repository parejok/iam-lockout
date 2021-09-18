# IAM User lockout based on failed logins

## Summary

This solution consists of a multi-account mechanism to lock an user account after a certain number of incorrect passwords in a pre-defined interval. The administrator will then have the option to unlock this user or reset his/her password.

## Architecture:

![architecture-overview](architecture-diagram/IAM-lockout.png)

## Installation Steps:

### Security Account (Stack):

- Login to your AWS Security Account, make sure you are in the us-east-1 region;

- Run the following commands to create Security Hub custom actions. The output of these commands will be the custom action ARNs, take note of them for later use;
```
aws securityhub create-action-target --name Unlock-IAM-User --description "This action unlocks the specified IAM User." --id UnlockIAMUser
aws securityhub create-action-target --name Reset-IAM-Password --description "This action updates the specified IAM User Password (which has to be changed in the first login)." --id UpdateIAMUserPasswd
```

- Deploy the template `IAM_user_lockout-master.yaml` in the us-east-1 region. The parameters:
	- pUnlockIAMActionArn: The custom action ARN for IAM Unlock Action (id=UnlockIAMUser);
	- pResetIAMPasswdActionArn: Custom Action ARN for IAM Reset Password Action (id=UpdateIAMUserPasswd);
	- pAdminEmail: The Administrator's e-mail, where the temporary passwords will be send to in case of an user password reset;

### Member Accounts (StackSets):

- Login to your Management Account (from where StackSets are deployed), make sure you are in the us-east-1 region;

- Deploy a new StackSet based on the template `IAM_user_lockout-members.yaml` in all accounts that will have the IAM user lockout feature implemented. Parameters:
  - pCentralAccountID: AccountID of Central Account (Security Hub delegated administrator);
  - pFailedLoginThreshold: Number of failed attempts before the user lockout;
  - pFailedLoginInterval: Interval of time (in minutes) where fails must occur to generate the user lockout.
  - pSuccessfulLogin: Choose this option if you want that 1 successful login resets the failed login counting.


## Process

### IAM user failed login:
1) User tries to login to AWS Management Console;
2) In the Authentication process, the entered password doesn't match and the login fails;
3) The failure sign-in attempt is recorded in CloudTrail;
4) The `ConsoleLogin` event triggers a EventBridge event;
5) EventBridge calls the Lambda Function `IAMLockout-BlockUser-Function`, that will verify:
	- If it's an IAM user (the solution doesn't block roles);
	- If there is any current block for this user;
	- If the number of attempts and interval match/exceed the parameters defined during the deployment;
   If the user matches all the requirements and there is no current block for this account, lambda will proceed to the account lockout (steps 6, 7, 8 and 9);
6) This Lambda function will add this login failure occurrence to the `LoginRegistryTable` table (from where it queries the interval/attempt thresholds)
7) It will then attach the `AWSDenyAll` policy to this user;
8) It will also create an entry on the `BlockedUserTable` DynamoDB table to ensure that there won't be duplicated entries for the same user.
9) Finally, Lambda will create the `IAM user exceeeded failed login attempts` finding on Security Hub for this user;
10) The member account's Security Hub will replicate the finding to the security account's Security Hub.
11) The administrator opens the Security Hub console from the security account to investigate the user lockout;

### Unlocking the IAM User:
12) The administrator decides to unlock this IAM user by selecting the finding on Security Hub and choosing the custom action `Unlock-IAM-User`;
13) By choosing this custom action, an EventBridge event will call the Lambda function `IAMLockout-UnlockUser-Function`, which will update the IAM user and DynamoDB table in the target account;
14) The function will detach the `AWSDenyAll` policy from the user;
15) The entry related to this user will be deleted from the `BlockedUserTable` table;
16) All entries related to this user's login attemps will also be deleted from the `LoginRegistryTable` table.

### Resetting the IAM User's Password:
17) The administrator decides to reset this IAM user's password by selecting the finding on Security Hub and choosing the custom action `Reset-IAM-Password`, which will update the IAM user and DynamoDB table in the target account;;
18) By choosing this custom action, an EventBridge event will call the Lambda function `IAMLockout-ResetPassword-Function`;
19) It will firstly call Secrets Manager, which will generate a temporary password;
20) The function will them update the IAM User password to this new temporary password, which must be changed after the first login;
21) A SNS notification will be sent to the administrator's e-mail containing this temporary password, which can be shared with the user;
22) The function will detach the `AWSDenyAll` policy from the user;
23) The entry related to this user will be deleted from the `BlockedUserTable` table;
24) All entries related to this user's login attemps will also be deleted from the `LoginRegistryTable` table.

### Lambda Prune
25) Every 24h EventBridge will trigger a scheduled event which will call a Lambda function;
26) This Lambda function will delete all entries older than 24h from the `LoginRegistryTable` table.

### Successful Login (Optional)
- If the parameter `pSuccessfulLogin` is set to `Yes` during the `IAM_user_lockout-members.yaml` deployment, a successful login will reset the counting of failed logins for the threshold. Please note that if the user has already been blocked (met the threshold), for safety reasons he/she will continue blocked even if afterwards the user logins in successfuly.
	
	