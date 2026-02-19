
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

# check the status

``` powershell
$allJobStatuses.Logs | ConvertFrom-Json
```

``` output
Event               : JobWarning
JobId               : 17c641fc-af2a-4c1d-a4d2-2951bd433030
Time                : 02/18/2026 23:23:52.333
TotalRetryCount     : 0
MigrationType       : Copy
MigrationDirection  : Import
ObjectType          : ListItem
Url                 : Documents/toOneDrive/folder/folder/folder/folder/Home Folder/folder/folder/folder/folder/file.xlsx
Id                  : e667f7ef-f565-4ea9-a602-a568cd9bedd4
SourceListItemIntId : 20051
TargetListItemIntId : 0
Message             : Field xd_Signature cannot be found. Field value will not be imported
CorrelationId       : dc50f8a1-1021-b000-daf7-3f0e9e9114be

Event                               : JobProgress
JobId                               : 17c641fc-af2a-4c1d-a4d2-2951bd433030
Time                                : 02/18/2026 23:24:06.756
FilesCreated                        : 1572
BytesProcessed                      : 2079292904
ObjectsProcessed                    : 3453
TotalExpectedSPObjects              : 3454
TotalErrors                         : 0
TotalWarnings                       : 676
TotalRetryCount                     : 0
MigrationType                       : Copy
MigrationDirection                  : Import
WaitTimeOnSqlThrottlingMilliseconds : 0
TotalDurationInMs                   : 0
CpuDurationInMs                     : 0
SqlDurationInMs                     : 0
SqlQueryCount                       : 0
IsShallowCopy                       : False
CreatedOrUpdatedFileStatsBySize     : {"10-100K":{"Count":500,"TotalSize":19331247,"TotalDownloadTime":145313,"TotalCreationTime":308018},"100K-1M":{"Count":230,"TotalSize":80924721,"TotalDownloadTime":56870,"TotalCreationTime":154709},"0-1K":{"Count":263,"TotalSize":65303,"TotalDownloadTime":97207,"TotalCreationTi
                                      me":159438},"1-10K":{"Count":19,"TotalSize":58634,"TotalDownloadTime":5122,"TotalCreationTime":8289},"1-10M":{"Count":546,"TotalSize":1731649493,"TotalDownloadTime":142588,"TotalCreationTime":754390},"10-100M":{"Count":11,"TotalSize":129343753,"TotalDownloadTime":2424,"TotalCre
                                      ationTime":36627}}
ObjectsStatsByType                  : {"SPUser":{"Count":5,"TotalTime":0,"AccumulatedVersions":0,"ObjectsWithVersions":0},"SPFolder":{"Count":154,"TotalTime":78696,"AccumulatedVersions":0,"ObjectsWithVersions":0},"SPListItem":{"Count":1726,"TotalTime":1188921,"AccumulatedVersions":0,"ObjectsWithVersions":0},"SPFile
                                      ":{"Count":1573,"TotalTime":1881840,"AccumulatedVersions":0,"ObjectsWithVersions":0}}
TotalExpectedBytes                  : 2079292904
BlobCopyFileStatesBySize            : {"10-100M":{"Count":3,"TotalSize":117919753,"TotalDownloadTime":1406,"TotalCreationTime":1128}}
LastSPObjectId                      : 1bed6e80-b540-4943-8f10-78a04b661585
CorrelationId                       : dc50f8a1-1021-b000-daf7-3f0e9e9114be

Event                               : JobEnd
JobId                               : 17c641fc-af2a-4c1d-a4d2-2951bd433030
Time                                : 02/18/2026 23:24:08.052
FilesCreated                        : 1573
BytesProcessed                      : 2079292904
ObjectsProcessed                    : 3454
TotalExpectedSPObjects              : 3454
TotalErrors                         : 0
TotalWarnings                       : 676
TotalRetryCount                     : 0
MigrationType                       : Copy
MigrationDirection                  : Import
WaitTimeOnSqlThrottlingMilliseconds : 0
TotalDurationInMs                   : 3166820
CpuDurationInMs                     : 672318
SqlDurationInMs                     : 940604
SqlQueryCount                       : 60858
IsShallowCopy                       : False
CreatedOrUpdatedFileStatsBySize     : {"10-100K":{"Count":500,"TotalSize":19331247,"TotalDownloadTime":145313,"TotalCreationTime":308018},"100K-1M":{"Count":230,"TotalSize":80924721,"TotalDownloadTime":56870,"TotalCreationTime":154709},"0-1K":{"Count":264,"TotalSize":65303,"TotalDownloadTime":97392,"TotalCreationTi
                                      me":159819},"1-10K":{"Count":19,"TotalSize":58634,"TotalDownloadTime":5122,"TotalCreationTime":8289},"1-10M":{"Count":546,"TotalSize":1731649493,"TotalDownloadTime":142588,"TotalCreationTime":754390},"10-100M":{"Count":11,"TotalSize":129343753,"TotalDownloadTime":2424,"TotalCre
                                      ationTime":36627}}
ObjectsStatsByType                  : {"SPUser":{"Count":5,"TotalTime":0,"AccumulatedVersions":0,"ObjectsWithVersions":0},"SPFolder":{"Count":154,"TotalTime":78696,"AccumulatedVersions":0,"ObjectsWithVersions":0},"SPListItem":{"Count":1727,"TotalTime":1189657,"AccumulatedVersions":0,"ObjectsWithVersions":0},"SPFile
                                      ":{"Count":1573,"TotalTime":1882408,"AccumulatedVersions":0,"ObjectsWithVersions":0}}
TotalExpectedBytes                  : 2079292904
BlobCopyFileStatesBySize            : {"10-100M":{"Count":3,"TotalSize":117919753,"TotalDownloadTime":1406,"TotalCreationTime":1128}}
CorrelationId                       : dc50f8a1-1021-b000-daf7-3f0e9e9114be
```
