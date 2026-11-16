Helper columns

Column	Formula	Meaning
callerAcct	=MID([@caller],14,12)	account the calling principal belongs to
roleAcct	=MID([@roleArn],14,12)	account that owns the assumed role
direction	=IF([@svc]="svc","svc",IF([@callerAcct]<>[@account],"INBOUND",IF([@roleAcct]<>[@account],"OUTBOUND","local")))	who crossed the account boundary
svc	=IF(ISNUMBER(SEARCH("amazonaws.com",[@sourceIp])),"svc","")	AWS service noise
localTime	=DATEVALUE(LEFT([@time],10))+TIMEVALUE(MID([@time],12,8))-5/24	Montreal time
compromised	=IF(COUNTIF(Keys!A:A,[@callerKey])>0,"MATCH","")	caller used a listed key
Put svc before direction in the table so the formula can reference it.

Hide the noise first. Filter out callerType = AWSService, sessionName starting with i- (EC2 instance profiles refreshing), and roleArn containing aws-service-role. What remains is usually a few hundred rows instead of hundreds of thousands.

Then filter, in this order:

compromised = MATCH. Confirmed attacker rows. Every roleArn is a role they reached, every issuedKey a credential they held.
callerType = IAMUser. Pivot on caller. Compare the user names with the IAM user list Task 1 is producing. A user you do not recognize was created by the attacker as persistence.
direction = OUTBOUND. Staging principals assuming roles in other accounts. Read roleAcct. Production (379293151802) or Stream Native (706912998770) is the critical finding of the whole incident.
direction = INBOUND. Outside principals assuming roles in staging. callerAcct that is not one of your organization’s accounts is either a vendor integration you know about or an attacker’s own AWS account. Check the role’s trust policy for anything you do not recognize.
svc blank, sourceIp not office, VPN or CI. Look the strangers up.
error = AccessDenied. Sort by caller. Bursts across many roles is enumeration.
New combinations. Make a pivot with rows caller and roleArn, values count of id and min of localTime. Sort by min descending. Legitimate automation starts on Nov 10 at midnight and runs every day. A pairing that first appears mid-window is new and needs an owner.
The usual tells, on what is left: durationSec > 3600, hours before 7 or after 19 local, roleArn containing Admin or OrganizationAccountAccessRole, rare userAgent and sessionName values from a pivot.
On a hit: filter callerKey = that row’s issuedKey to follow what the session did next inside this account, and send the key to the Task 1 and Task 2 people so they trace it in theirs.

One scope note for the report: this filter covers AssumeRole only. SSO logins and GitHub-style OIDC appear as AssumeRoleWithSAML and AssumeRoleWithWebIdentity, which Volodymyr did not ask for. Say so in the scope line so nobody assumes they were checked.

Skipped: no formulas for the two other accounts, repeat the same sheet per account. Risk: rows with a blank roleArn or a service caller produce junk in the ARN helper columns, which is why direction checks svc first.

