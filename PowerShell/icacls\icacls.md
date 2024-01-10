Use TAKEOWN and ICACLS with very long paths and filenames

$iexcmd = 'TAKEOWN /F ' + '"<folder / drive letter>" /A /R /D Y > c:\takeown_log.log 2>&1'
iex $iexcmd | Out-Null

$iexcmd = 'ICACLS ' + '"\\?\<driveLetter>:<path>" /q /c /t /reset > c:\failed_perms.log 2>&1'
iex $iexcmd | Out-Null
ref
https://notes.ponderworthy.com/use-takeown-and-icacls-with-very-long-paths-and-filenames
