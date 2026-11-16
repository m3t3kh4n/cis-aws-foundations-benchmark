At that size jq’s normal parser is the problem too. It builds the whole 800 MB object in memory, which needs roughly 6 to 8 GB of RAM per file. jq has a streaming parser for exactly this. It walks the file as a sequence of path and value events and only ever holds one CloudTrail event in memory. Everything downstream must stay streaming as well, so no Group-Object or Sort-Object on the full CSV.

1. Streaming filter. Save as sts_stream.jq:

fromstream(2 | truncate_stream(inputs)) | .CloudTrailEvent | fromjson |
[(.sharedEventID // .eventID), .recipientAccountId, .awsRegion, .eventTime, .eventName, .userIdentity.type,
 (.userIdentity.arn // .userIdentity.invokedBy // ""), (.userIdentity.accessKeyId // ""),
 (.sourceIPAddress // ""), (.userAgent // ""),
 (.requestParameters.roleArn // ""), (.requestParameters.roleSessionName // .requestParameters.name // ""),
 (.requestParameters.durationSeconds // ""),
 (.responseElements.credentials.accessKeyId // ""), (.responseElements.credentials.expiration // ""),
 (.responseElements.assumedRoleUser.arn // .responseElements.federatedUser.arn // ""),
 (.errorCode // "")] | @csv
The first line strips the Events array wrapper and rebuilds each element one at a time. The rest is unchanged.

2. Same loop as before, with -n --stream:

$csv = "$PWD\sts_audit.csv"
'"id","account","region","time","event","callerType","caller","callerKey","sourceIp","userAgent","roleArn","sessionName","durationSec","issuedKey","expires","resultArn","error"' | Set-Content $csv -Encoding ascii
foreach ($f in Get-ChildItem *.json | Where-Object Length -gt 0) {
  Write-Host "$($f.Name)  $([math]::Round($f.Length/1MB)) MB"
  cmd /c "jq -rn --stream -f sts_stream.jq `"$($f.FullName)`" >> `"$csv`""
  if ($LASTEXITCODE -ne 0) { Write-Warning "jq failed on $($f.Name)" }
}
Memory stays flat at a few tens of megabytes no matter how big the file is. The streaming parser is slower, so expect a few minutes per 800 MB file. Start it and leave it.

3. Post-processing that never loads the whole CSV.

Human and key-based callers only, which is the list the report needs row by row. -ReadCount processes 10,000 lines at a time:

Get-Content sts_audit.csv -Encoding utf8 -ReadCount 10000 | ForEach-Object { $_ -notmatch '"AWSService"' } | Set-Content sts_human.csv -Encoding utf8
Dedupe only that small file, not the big one:

$seen = [Collections.Generic.HashSet[string]]::new()
Get-Content sts_human.csv -Encoding utf8 | Where-Object { $seen.Add($_.Split(',')[0]) } | Set-Content sts_human_dedup.csv -Encoding utf8
Summary counts per account, event and caller type for the Confluence table, streamed with a hashtable:

$c = @{}
Get-Content sts_audit.csv -Encoding utf8 -ReadCount 10000 | ForEach-Object { foreach ($l in $_) { $c[($l.Split(',')[1,4,5] -join ' ')]++ } }
$c.GetEnumerator() | Sort-Object Name | Format-Table Name, Value -AutoSize
The split on commas is safe for those three columns because nothing before the user agent column can contain a comma. Ignore the one row produced by the header line.

For the Confluence page attach sts_human_dedup.csv and the summary table. The full multi-gigabyte CSV goes in a shared drive with a link, not as a page attachment.

Skipped: untested at this scale here. Risk: the summary counts include cross-account calls twice, once per account, so say “events as recorded per account” in the table caption rather than deduped totals.

