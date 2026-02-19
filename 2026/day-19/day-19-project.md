# Task 1: Log Rotation Script
```bash
#!/bin/bash


<< log
# log_rotate.sh
# Usage:
 ./log_rotate.sh <path to your source> <path to backup folder>
log

function display_usage {
        echo "./log_rotate.sh <path to your source> <path to backup folder>"
}

if [ $# -eq 0 ]; then
        display_usage
fi
source_dir=$1
timestamp=$(date '+%y-%m-%d-%H-%M-%S')
backup_dir=$2
function create_backup {
        zip -r "${backup_dir}/backup_${timestamp}.zip" "${source_dir}" >/dev/null
        echo " backup genrated successfully for ${timestamp} "

}

function perform_rotation {
        backup=($(ls -t "${backup_dir}/backup_"*.zip 2>/dev/null ))
        echo "${#backup[@]}"    

        if [ "${#backup[@]}" -gt 5 ]; then
                echo "performing rotation for five days"

        backup_to_remove=("${backup[@]:5}")
        echo "${backup_to_remove[@]}"
        for backup in "${backup_to_remove[@]}";
        do
                rm -f ${backup}
        done
        fi
}
create_backup
perform_rotation

```

# Task 2: Server Backup Script
```bash 
#!/bin/bash
<< log
# log_rotate.sh
# Usage:
# ./log_rotate.sh <source_directory> <backup_directory>
log
set -e

display_usage() {
    echo "Usage: $0 <source_directory> <backup_directory>"
    exit 1
}

# Check arguments
if [ $# -ne 2 ]; then
    display_usage
fi

source_dir="$1"
backup_dir="$2"

# Check if source exists
if [ ! -d "$source_dir" ]; then
    echo "Error: Source directory does not exist!"
    exit 1
fi

# Create backup directory if it doesn't exist
mkdir -p "$backup_dir"

# Create timestamp
timestamp=$(date '+%Y-%m-%d')
archive_name="backup-${timestamp}.tar.gz"
archive_path="${backup_dir}/${archive_name}"

echo "Creating backup..."

# Create tar.gz archive
tar -czf "$archive_path" -C "$(dirname "$source_dir")" "$(basename "$source_dir")"

# Verify archive creation
if [ -f "$archive_path" ]; then
    echo "Backup created successfully."

    # Get archive size
    archive_size=$(du -h "$archive_path" | cut -f1)

    echo "Archive Name: $archive_name"
    echo "Archive Size: $archive_size"
else
    echo "Backup failed!"
    exit 1
fi

echo "Deleting backups older than 14 days..."

# Delete backups older than 14 days
find "$backup_dir" -name "backup-*.tar.gz" -type f -mtime +14 -exec rm -f {} \;

echo "Old backups cleanup completed."

echo "Done."
```
# Task 3: Crontab
# Run log_rotate.sh every day at 2 AM
0 2 * * * /day_03/log_rotate.sh

# Run backup.sh every Sunday at 3 AM
0 3 * * 0 /day_03/backup.sh

# Run health_check.sh every 5 minutes
*/5 * * * * /var/log/health_check.sh



