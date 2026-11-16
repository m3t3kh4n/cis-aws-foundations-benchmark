Confirm which account you are in:

aws sts get-caller-identity --profile PROFILE

List the regions to loop over:

aws ec2 describe-regions --profile PROFILE --region us-east-1 --query 'Regions[].RegionName' --output text

AssumeRole calls since Nov 10:

aws cloudtrail lookup-events --profile PROFILE --region REGION --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --start-time 2026-11-10T00:00:00Z --output json > assumerole_PROFILE_REGION.json

GetSessionToken calls:

aws cloudtrail lookup-events --profile PROFILE --region REGION --lookup-attributes AttributeKey=EventName,AttributeValue=GetSessionToken --start-time 2026-11-10T00:00:00Z --output json > getsessiontoken_PROFILE_REGION.json

GetFederationToken calls:

aws cloudtrail lookup-events --profile PROFILE --region REGION --lookup-attributes AttributeKey=EventName,AttributeValue=GetFederationToken --start-time 2026-11-10T00:00:00Z --output json > getfederationtoken_PROFILE_REGION.json

The CLI paginates automatically, so each file holds every event for that account, region and event type. The API allows 50 events per page at 2 requests per second, so a busy production region can take several minutes. Each event’s CloudTrailEvent field is a JSON string containing the full record, including responseElements.credentials.accessKeyId and expiration, which are the temporary credentials issued.

Quick look at one file without jq, using only the CLI’s own query language:

aws cloudtrail lookup-events --profile PROFILE --region REGION --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --start-time 2026-11-10T00:00:00Z --query 'Events[].[EventTime,Username,EventId]' --output table

Flatten all the files to one CSV once collected:

{ echo 'account,region,time,event,callerType,caller,callerKey,sourceIp,userAgent,roleArn,sessionName,durationSec,issuedKey,expires,resultArn,error'; jq -r '.Events[].CloudTrailEvent | fromjson | [.recipientAccountId,.awsRegion,.eventTime,.eventName,.userIdentity.type,(.userIdentity.arn // .userIdentity.invokedBy // ""),(.userIdentity.accessKeyId // ""),.sourceIPAddress,.userAgent,(.requestParameters.roleArn // ""),(.requestParameters.roleSessionName // .requestParameters.name // ""),(.requestParameters.durationSeconds // ""),(.responseElements.credentials.accessKeyId // ""),(.responseElements.credentials.expiration // ""),(.responseElements.assumedRoleUser.arn // .responseElements.federatedUser.arn // ""),(.errorCode // "")] | @csv' *.json; } > sts_audit.csv

Only the human or key-based callers, which is where anything suspicious will be:

jq -r '.Events[].CloudTrailEvent | fromjson | select(.userIdentity.type != "AWSService") | [.eventTime,.recipientAccountId,.eventName,.userIdentity.arn,.userIdentity.accessKeyId,.sourceIPAddress,.requestParameters.roleArn,.responseElements.credentials.accessKeyId] | @tsv' *.json

Skipped: no per-region loop wrapper, since you asked for bare commands. Risk: lookup-events reads only the last 90 days and nothing from regions where CloudTrail is off, and the Nov 10 start is UTC midnight.
