Here is the hourly version. It writes one file per hour, saves each file only when its hour is complete, and skips finished hours on rerun, so you can stop and restart it as often as you like until every hour is in.

$P = "staging"; $R = "ca-central-1"; $E = "AssumeRole"
$env:AWS_RETRY_MODE = "adaptive"; $env:AWS_MAX_ATTEMPTS = "25"
$start = [datetime]::Parse("2026-11-10T00:00:00Z").ToUniversalTime()
$end   = [datetime]::UtcNow
$chunk = [timespan]::FromHours(1)
$dir   = "$PWD\chunks_${P}_${R}_${E}"; New-Item -ItemType Directory -Force $dir | Out-Null

for ($t = $start; $t -lt $end; $t += $chunk) {
  $t2 = $t + $chunk; if ($t2 -gt $end) { $t2 = $end }
  $done = Join-Path $dir ("{0:yyyyMMdd_HHmm}.jsonl" -f $t); $part = "$done.part"
  if (Test-Path $done) { continue }
  $w = [IO.StreamWriter]::new($part, $false, [Text.UTF8Encoding]::new($false))
  $token = $null; $n = 0; $ok = $true
  do {
    $a = @('cloudtrail','lookup-events','--profile',$P,'--region',$R,
           '--lookup-attributes',"AttributeKey=EventName,AttributeValue=$E",
           '--start-time',($t.ToString('s') + 'Z'),'--end-time',($t2.ToString('s') + 'Z'),
           '--max-items','50','--output','json')
    if ($token) { $a += @('--starting-token', $token) }
    $tries = 0
    do { $out = aws @a; $tries++; if ($LASTEXITCODE -ne 0) { Start-Sleep -Seconds (10 * $tries) } } while ($LASTEXITCODE -ne 0 -and $tries -lt 5)
    if ($LASTEXITCODE -ne 0) { $ok = $false; break }
    $page = ($out -join "`n") | ConvertFrom-Json
    foreach ($ev in $page.Events) { $w.WriteLine($ev.CloudTrailEvent); $n++ }
    $token = $page.NextToken
    Start-Sleep -Milliseconds 600
  } while ($token)
  $w.Close()
  if ($ok) { Move-Item $part $done -Force; Write-Host ("{0}Z  {1} events" -f $t.ToString('s'), $n) }
  else     { Remove-Item $part; Write-Warning ("FAILED {0}Z, rerun to resume" -f $t.ToString('s')) }
}
What changed from the earlier version:

One page at a time with a 0.6 second pause, so you stay under the 2 per second limit by yourself. A throttled page is retried up to 5 times with growing waits before the hour is marked failed.
Events are written to disk as they arrive, one per line, instead of being collected in memory. A half-finished hour is a .part file and is thrown away, so a file ending in .jsonl is always complete.
The JSON lines are joined before parsing, which Windows PowerShell 5 needs for multi-line CLI output.
Check completeness when it finishes. The file count must equal the number of hours in the window and there must be no .part left:

(Get-ChildItem $dir -Filter *.jsonl).Count; [math]::Ceiling(($end - $start).TotalHours); Get-ChildItem $dir -Filter *.part
Convert to CSV. Each line is already a complete event, so jq reads them one at a time with flat memory, and no --stream is needed. Save this as sts_jsonl.jq:

[(.sharedEventID // .eventID), .recipientAccountId, .awsRegion, .eventTime, .eventName, .userIdentity.type,
 (.userIdentity.arn // .userIdentity.invokedBy // ""), (.userIdentity.accessKeyId // ""),
 (.sourceIPAddress // ""), (.userAgent // ""),
 (.requestParameters.roleArn // ""), (.requestParameters.roleSessionName // .requestParameters.name // ""),
 (.requestParameters.durationSeconds // ""),
 (.responseElements.credentials.accessKeyId // ""), (.responseElements.credentials.expiration // ""),
 (.responseElements.assumedRoleUser.arn // .responseElements.federatedUser.arn // ""),
 (.errorCode // "")] | @csv
Append all hours to the CSV that already has the header from the other regions:

foreach ($f in Get-ChildItem .\chunks_*\*.jsonl) { cmd /c "jq -r -f sts_jsonl.jq `"$($f.FullName)`" >> `"$csv`"" }
An event sitting exactly on an hour boundary can land in two chunks. The id column dedupe from before removes it.

Expect about 80 events per second, so 500,000 events take under two hours. Run it for AssumeRole, then change $E and rerun for the other two names, which finish in a minute since they are empty.

Skipped: untested here. Risk: if your SSO session expires mid-run, every hour after that fails fast, so run aws sso login --sso-session intelcom and rerun the block, it picks up where it stopped.

