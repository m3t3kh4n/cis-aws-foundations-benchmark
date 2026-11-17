Three things can make it fail, and I can’t see which one hit you: the filter file was saved with a byte order mark, which jq reads as a syntax error at line 1; an earlier filter that expects the .Events[] wrapper was used on the line-per-event files; or $csv was not defined in a fresh PowerShell window, so the >> had no target. The recipe below removes all three.

1. Create the filter from PowerShell, so it is plain ASCII with no byte order mark. This version accepts any of the formats you have produced: a JSON object per line, a quoted JSON string per line, or the old {"Events": [...]} files.

@'
(if type == "object" and has("Events") then .Events[].CloudTrailEvent else . end)
| (if type == "string" then fromjson else . end)
| [(.sharedEventID // .eventID), .recipientAccountId, .awsRegion, .eventTime, .eventName, .userIdentity.type,
   (.userIdentity.arn // .userIdentity.invokedBy // ""), (.userIdentity.accessKeyId // ""),
   (.sourceIPAddress // ""), (.userAgent // ""),
   (.requestParameters.roleArn // ""), (.requestParameters.roleSessionName // .requestParameters.name // ""),
   (.requestParameters.durationSeconds // ""),
   (.responseElements.credentials.accessKeyId // ""), (.responseElements.credentials.expiration // ""),
   (.responseElements.assumedRoleUser.arn // .responseElements.federatedUser.arn // ""),
   (.errorCode // "")] | @csv
'@ | Set-Content sts_any.jq -Encoding ascii
2. Test on one file before looping. The first command shows how a line starts, the second should print three CSV rows.

$one = (Get-ChildItem .\chunks_*\*.jsonl | Where-Object Length -gt 0 | Select-Object -First 1).FullName
Get-Content $one -TotalCount 1 | ForEach-Object { $_.Substring(0, [Math]::Min(80, $_.Length)) }
jq -r -f sts_any.jq $one | Select-Object -First 3
A line beginning with {"eventVersion" is a normal object. One beginning with "{\"eventVersion\" is a quoted string, and the second line of the filter now handles that too.

3. Run the loop with $csv defined in the same window:

$csv = "$PWD\sts_audit.csv"
if (-not (Test-Path $csv)) { '"id","account","region","time","event","callerType","caller","callerKey","sourceIp","userAgent","roleArn","sessionName","durationSec","issuedKey","expires","resultArn","error"' | Set-Content $csv -Encoding ascii }
foreach ($f in Get-ChildItem .\chunks_*\*.jsonl | Where-Object Length -gt 0) {
  cmd /c "jq -r -f sts_any.jq `"$($f.FullName)`" >> `"$csv`""
  if ($LASTEXITCODE -ne 0) { Write-Warning "jq failed on $($f.FullName)" }
}
Memory stays flat on the line-per-event files because jq handles each line as its own input. Keep the --stream version for the old 800 MB single-object files only.

If step 2 still errors, paste the first line jq prints. It names the file, the line number and what it found, which pins the cause immediately.

Skipped: not run here, no jq or sample files on this side. Risk: if a chunk’s .jsonl was written by a different script than the one I gave, the column order inside each line could differ, but the filter reads by field name so that does not matter.

