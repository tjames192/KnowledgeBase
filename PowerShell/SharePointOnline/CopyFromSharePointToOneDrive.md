
``` powershell
# setup the source and login
$env:ENTRAID_APP_ID = "00000000-0000-0000-0000-0000000000000"
$fromSharepoint = "https://tenant.sharepoint.com/sites/site"
$Tenant = "00000000-0000-0000-0000-0000000000000"
$Connection = Connect-PnPOnline -Url $fromSharepoint -ReturnConnection -Tenant $Tenant -DeviceLogin

# target destination
$toOneDrive = "https://tenant-my.sharepoint.com/personal/user_tenant_com/Documents/toOneDrive"

# get each folder in "Shared Documents", note we are not getting each subdirectory. just the top folder
$fromAllFolders = Get-PnPFolderInFolder -Identity "Shared Documents" -Connection $Connection
# skip system folders
$fromAllFolders = $fromAllFolders.where({$_.Name -notmatch "Forms"})

# create background job for each folder
$allJobs = foreach ($fromFolder in $fromAllFolders) {
    $sourceUrl = $fromFolder.ServerRelativeUrl
    # Copy-PnPFile will copy subfolders and files
    $job = Copy-PnPFile -SourceUrl $sourceUrl -TargetUrl $toOneDrive -Overwrite -Force -IgnoreVersionHistory -AllowSchemaMismatch -NoWait -Connection $Connection
    $job
}

# get statuses for each job in parallel
# Install-Module SplitPipeline -Scope AllUsers
$allJobStatuses = $allJobs | Split-Pipeline -count 5 -Module "PnP.PowerShell" -Variable Connection {
    process {
        $job = $_
        # keep checking over n over until JobState = 0 then break loop, else sleep 1min for each check
        while($true) {
            $jobStatus = Receive-PnPCopyMoveJobStatus -Job $job -Connection $Connection
            
            if($jobStatus.JobState -eq 0) {
                $jobState = "Job finished {0}" -f $job.JobId
                write-verbose -verbose $jobState
                $jobStatus
                break
            }
            start-sleep 60
        }
    }
}

```
