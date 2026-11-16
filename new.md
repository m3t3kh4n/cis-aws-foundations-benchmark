1. Save this as sts.jq in the folder with your JSON files:

(["account","region","time","event","callerType","caller","callerKey","sourceIp","userAgent","roleArn","sessionName","durationSec","issuedKey","expires","resultArn","error"] | @csv),
([inputs.Events[].CloudTrailEvent | fromjson]
 | unique_by(.sharedEventID // .eventID) | sort_by(.eventTime) | .[]
 | [.recipientAccountId, .awsRegion, .eventTime, .eventName, .userIdentity.type,
    (.userIdentity.arn // .userIdentity.invokedBy // ""), (.userIdentity.accessKeyId // ""),
    (.sourceIPAddress // ""), (.userAgent // ""),
    (.requestParameters.roleArn // ""), (.requestParameters.roleSessionName // .requestParameters.name // ""),
    (.requestParameters.durationSeconds // ""),
    (.responseElements.credentials.accessKeyId // ""), (.responseElements.credentials.expiration // ""),
    (.responseElements.assumedRoleUser.arn // .responseElements.federatedUser.arn // ""),
    (.errorCode // "")]
 | @csv)

2. Run it from that folder:

$OutputEncoding = [Console]::OutputEncoding = [Text.UTF8Encoding]::new($false)
Get-Content *.json -Raw -Encoding utf8 | jq -rn -f sts.jq | Set-Content sts_audit.csv -Encoding utf8

The first line stops PowerShell from corrupting non-ASCII characters on the way in and out of jq. The output is one CSV for every account and region, deduplicated on the shared event ID so cross-account calls appear once, sorted by time, and ready for Excel or a Confluence table.

If jq is not installed on that machine, this does the same thing with nothing but PowerShell and leaves the rows in $rows for filtering:

$rows = foreach ($f in Get-ChildItem *.json | Where-Object Length -gt 0) {
  foreach ($ev in (Get-Content $f -Raw -Encoding utf8 | ConvertFrom-Json).Events) {
    $e = $ev.CloudTrailEvent | ConvertFrom-Json
    [pscustomobject]@{
      account = $e.recipientAccountId; region = $e.awsRegion; time = $e.eventTime; event = $e.eventName
      callerType = $e.userIdentity.type
      caller = if ($e.userIdentity.arn) { $e.userIdentity.arn } else { $e.userIdentity.invokedBy }
      callerKey = $e.userIdentity.accessKeyId; sourceIp = $e.sourceIPAddress; userAgent = $e.userAgent
      roleArn = $e.requestParameters.roleArn
      sessionName = if ($e.requestParameters.roleSessionName) { $e.requestParameters.roleSessionName } else { $e.requestParameters.name }
      durationSec = $e.requestParameters.durationSeconds
      issuedKey = $e.responseElements.credentials.accessKeyId; expires = $e.responseElements.credentials.expiration
      resultArn = if ($e.responseElements.assumedRoleUser.arn) { $e.responseElements.assumedRoleUser.arn } else { $e.responseElements.federatedUser.arn }
      error = $e.errorCode
      id = if ($e.sharedEventID) { $e.sharedEventID } else { $e.eventID }
    }
  }
}
$rows = $rows | Sort-Object id -Unique | Sort-Object time
$rows | Export-Csv sts_audit.csv -NoTypeInformation -Encoding UTF8

Then the two views you need for the report come straight from $rows:

$rows | Group-Object account, event, callerType | Sort-Object Name | Format-Table Count, Name -AutoSize
$rows | Where-Object callerType -ne 'AWSService' | Export-Csv sts_human.csv -NoTypeInformation -Encoding UTF8

The first is the summary table for Confluence. The second is the short list of every call made by a person or a long-lived key, which is where the compromised staging keys will show up if they were used.

Skipped: not run on real files, since I have none here. Risk: Windows PowerShell 5 is slow on very large JSON, so if a production region file is tens of megabytes the pure PowerShell path can take a few minutes, while jq will not.
