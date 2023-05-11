# low free space in disk partition

## Recommended Actions

Free space in /shared , /var parition(s)

1. Log in to the command line.
2. Check disk space usage with command in bash:

        df -h
        
3. Run the below command to list the top 10 largest files in /shared , /var partition(s)

        find /shared/ -printf '%s %p\n'| sort -nr | head -10
        find /var/ -printf '%s %p\n'| sort -nr | head -10
        
4. Remove large files which are not necessary and free space

        find /var/ -type f -name "*.ucs" -print -delete
        find /shared/ -type f -name "*.ucs" -print -delete
        find /shared/tmp -type f -name "rpm-tmp*" -print -delete
        find /shared/tmp -type f -name "sess_*" -mtime +1 -print -delete
        find /shared/tmp -type d -name "im*" -print -exec rm -rf {} +
        find /shared/tmp -type d -name "ts_db.save_dir_*" -print -exec rm -rf {} +
        
## Additional Information
https://my.f5.com/manage/s/article/K15585100
