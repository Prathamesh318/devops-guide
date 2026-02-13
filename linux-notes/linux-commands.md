# 🐧 Linux Commands - Complete DevOps Guide
## From Beginner to Advanced

---

# Table of Contents
1. [File System Navigation](#1-file-system-navigation)
2. [File Operations](#2-file-operations)
3. [File Permissions](#3-file-permissions)
4. [Text Processing](#4-text-processing)
5. [User Management](#5-user-management)
6. [Process Management](#6-process-management)
7. [Disk & Storage](#7-disk--storage)
8. [Networking](#8-networking)
9. [Package Management](#9-package-management)
10. [System Monitoring](#10-system-monitoring)
11. [Compression & Archives](#11-compression--archives)
12. [Shell Scripting Essentials](#12-shell-scripting-essentials)
13. [Advanced Commands](#13-advanced-commands)

---

# 1. File System Navigation

## `pwd` - Print Working Directory
Shows your current location in the filesystem.

```bash
pwd                    # Output: /home/username
pwd -L                 # Show logical path (with symlinks)
pwd -P                 # Show physical path (resolved symlinks)
```

## `cd` - Change Directory
Navigate between directories.

```bash
cd /path/to/dir        # Go to absolute path
cd folder              # Go to relative folder
cd ..                  # Go up one level
cd ../..               # Go up two levels
cd ~                   # Go to home directory
cd -                   # Go to previous directory
cd                     # Same as cd ~ (home)
```

## `ls` - List Directory Contents
View files and folders.

```bash
ls                     # Basic listing
ls -l                  # Long format (permissions, size, date)
ls -a                  # Show hidden files (starting with .)
ls -la                 # Long format + hidden files
ls -lh                 # Human-readable sizes (KB, MB, GB)
ls -lS                 # Sort by size (largest first)
ls -lt                 # Sort by time (newest first)
ls -ltr                # Sort by time (oldest first)
ls -R                  # Recursive listing (include subdirs)
ls -d */               # List only directories
ls *.txt               # List only .txt files
ls -i                  # Show inode numbers
ls --color=auto        # Colorized output
```

## `tree` - Directory Tree View
Visual representation of directory structure.

```bash
tree                   # Full tree from current dir
tree -L 2              # Limit depth to 2 levels
tree -d                # Directories only
tree -a                # Include hidden files
tree -h                # Show file sizes
tree -p                # Show permissions
tree -f                # Show full path
tree -I "node_modules" # Ignore pattern
```

---

# 2. File Operations

## `touch` - Create Empty Files
Create files or update timestamps.

```bash
touch file.txt              # Create new file
touch file1.txt file2.txt   # Create multiple files
touch -a file.txt           # Update access time only
touch -m file.txt           # Update modification time only
touch -d "2024-01-01" file  # Set specific date
touch -r ref.txt file.txt   # Copy timestamp from another file
```

## `mkdir` - Create Directories

```bash
mkdir folder                # Create single directory
mkdir dir1 dir2 dir3        # Create multiple directories
mkdir -p a/b/c/d            # Create nested directories
mkdir -m 755 folder         # Create with specific permissions
mkdir -v folder             # Verbose output
```

## `cp` - Copy Files and Directories

```bash
cp source.txt dest.txt      # Copy file
cp file.txt /path/to/dir/   # Copy to directory
cp -r source_dir/ dest_dir/ # Copy directory recursively
cp -i file.txt dest/        # Interactive (prompt before overwrite)
cp -n file.txt dest/        # Never overwrite
cp -u source dest           # Update only (copy if source is newer)
cp -v file.txt dest/        # Verbose output
cp -p file.txt dest/        # Preserve permissions and timestamps
cp -a source/ dest/         # Archive mode (preserve everything)
cp *.txt backup/            # Copy all .txt files
```

## `mv` - Move/Rename Files

```bash
mv old.txt new.txt          # Rename file
mv file.txt /path/to/dir/   # Move file
mv -i file.txt dest/        # Interactive (prompt before overwrite)
mv -n file.txt dest/        # Never overwrite
mv -u source dest           # Update only
mv -v file.txt dest/        # Verbose output
mv *.log logs/              # Move all .log files
```

## `rm` - Remove Files and Directories

```bash
rm file.txt                 # Remove file
rm -i file.txt              # Interactive (confirm before delete)
rm -f file.txt              # Force (no confirmation)
rm -r directory/            # Remove directory recursively
rm -rf directory/           # Force remove directory (DANGEROUS!)
rm -v file.txt              # Verbose output
rm *.tmp                    # Remove all .tmp files
rm -d empty_dir             # Remove empty directory
```

> ⚠️ **WARNING**: `rm -rf /` can destroy your entire system. Always double-check paths!

## `ln` - Create Links

```bash
ln source.txt hardlink.txt       # Create hard link
ln -s source.txt symlink.txt     # Create symbolic (soft) link
ln -sf source.txt symlink.txt    # Force create symlink
ln -s /path/to/dir link_name     # Link to directory
```

**Hard vs Soft Links:**
- **Hard Link**: Points to same inode; works if original deleted
- **Soft Link**: Points to filename; breaks if original deleted

## `cat` - Concatenate and Display Files

```bash
cat file.txt                # Display file content
cat file1.txt file2.txt     # Display multiple files
cat -n file.txt             # Show line numbers
cat -b file.txt             # Number non-blank lines only
cat -s file.txt             # Squeeze blank lines
cat > newfile.txt           # Create file (Ctrl+D to save)
cat >> file.txt             # Append to file
cat file1.txt file2.txt > merged.txt  # Merge files
```

## `less` & `more` - Page Through Files

```bash
less file.txt               # View file with navigation
more file.txt               # Simple pager

# less navigation:
# Space/f    - Next page
# b          - Previous page
# /pattern   - Search forward
# ?pattern   - Search backward
# n          - Next search result
# N          - Previous search result
# g          - Go to beginning
# G          - Go to end
# q          - Quit
```

## `head` & `tail` - View File Parts

```bash
head file.txt               # First 10 lines
head -n 20 file.txt         # First 20 lines
head -c 100 file.txt        # First 100 bytes

tail file.txt               # Last 10 lines
tail -n 20 file.txt         # Last 20 lines
tail -f file.txt            # Follow file (live updates)
tail -F file.txt            # Follow with retry (if file rotates)
tail -n +5 file.txt         # From line 5 to end
```

## `find` - Search for Files

```bash
find /path -name "file.txt"          # Find by exact name
find . -name "*.log"                 # Find by pattern
find . -iname "*.TXT"                # Case-insensitive
find . -type f                       # Find files only
find . -type d                       # Find directories only
find . -type l                       # Find symlinks
find . -size +10M                    # Files larger than 10MB
find . -size -1k                     # Files smaller than 1KB
find . -mtime -7                     # Modified in last 7 days
find . -mtime +30                    # Modified more than 30 days ago
find . -mmin -60                     # Modified in last 60 minutes
find . -empty                        # Empty files/directories
find . -perm 755                     # By exact permissions
find . -user username                # By owner
find . -group groupname              # By group
find . -name "*.tmp" -delete         # Find and delete
find . -name "*.sh" -exec chmod +x {} \;  # Execute command
find . -name "*.log" -exec rm {} +   # Efficient batch execution
find . -maxdepth 2 -name "*.txt"     # Limit search depth
find . -not -name "*.txt"            # Exclude pattern
find . -name "*.txt" -o -name "*.md" # OR condition
```

## `locate` - Fast File Search

```bash
locate filename             # Fast search (uses database)
locate -i filename          # Case-insensitive
locate -c "*.log"           # Count matches
locate -l 10 pattern        # Limit to 10 results
sudo updatedb               # Update locate database
```

## `which` & `whereis` - Find Commands

```bash
which python                # Find command path
which -a python             # All matching paths
whereis python              # Find binary, source, and man pages
type python                 # Show how command would be interpreted
```

---

# 3. File Permissions

## Understanding Permissions
```
-rwxr-xr-x  1  user  group  4096  Jan 29 10:00  file.txt
│└┬┘└┬┘└┬┘
│ │  │  └── Others: r-x (read, execute)
│ │  └───── Group:  r-x (read, execute)
│ └──────── Owner:  rwx (read, write, execute)
└────────── File type: - (file), d (directory), l (link)
```

**Permission Values:**
- `r` (read) = 4
- `w` (write) = 2
- `x` (execute) = 1

## `chmod` - Change Permissions

```bash
# Symbolic Mode
chmod u+x file.txt          # Add execute for user
chmod g-w file.txt          # Remove write for group
chmod o=r file.txt          # Set others to read only
chmod a+x file.txt          # Add execute for all
chmod u+rwx,g+rx,o+r file   # Multiple changes
chmod +x script.sh          # Add execute for all
chmod -x script.sh          # Remove execute for all

# Numeric Mode
chmod 755 file.txt          # rwxr-xr-x
chmod 644 file.txt          # rw-r--r--
chmod 700 file.txt          # rwx------
chmod 777 file.txt          # rwxrwxrwx (avoid this!)
chmod 600 private.key       # rw------- (common for SSH keys)

# Recursive
chmod -R 755 directory/     # Apply to all files in directory
chmod -R u+rw directory/    # Add read/write recursively
```

**Common Permission Patterns:**
| Value | Meaning | Use Case |
|-------|---------|----------|
| 755 | rwxr-xr-x | Scripts, directories |
| 644 | rw-r--r-- | Regular files |
| 600 | rw------- | Private files, SSH keys |
| 700 | rwx------ | Private directories |
| 777 | rwxrwxrwx | Avoid! Security risk |

## `chown` - Change Ownership

```bash
chown user file.txt              # Change owner
chown user:group file.txt        # Change owner and group
chown :group file.txt            # Change group only
chown -R user:group directory/   # Recursive
chown --reference=ref.txt file   # Copy ownership from another file
```

## `chgrp` - Change Group

```bash
chgrp group file.txt             # Change group
chgrp -R group directory/        # Recursive
```

## Special Permissions

```bash
# SUID (Set User ID) - Execute as file owner
chmod u+s file                   # Or chmod 4755 file

# SGID (Set Group ID) - Execute as group owner
chmod g+s file                   # Or chmod 2755 file

# Sticky Bit - Only owner can delete in directory
chmod +t directory               # Or chmod 1755 directory
```

---

# 4. Text Processing

## `grep` - Search Text Patterns

```bash
grep "pattern" file.txt          # Basic search
grep -i "pattern" file.txt       # Case-insensitive
grep -v "pattern" file.txt       # Invert (lines NOT matching)
grep -n "pattern" file.txt       # Show line numbers
grep -c "pattern" file.txt       # Count matches
grep -l "pattern" *.txt          # List files with matches
grep -L "pattern" *.txt          # List files without matches
grep -r "pattern" directory/     # Recursive search
grep -w "word" file.txt          # Match whole word only
grep -A 3 "pattern" file.txt     # 3 lines after match
grep -B 3 "pattern" file.txt     # 3 lines before match
grep -C 3 "pattern" file.txt     # 3 lines before and after
grep -E "regex" file.txt         # Extended regex (egrep)
grep -P "perl-regex" file.txt    # Perl regex
grep "^start" file.txt           # Lines starting with "start"
grep "end$" file.txt             # Lines ending with "end"
grep "pattern1\|pattern2" file   # Multiple patterns (OR)
grep -e "pat1" -e "pat2" file    # Multiple patterns
```

## `sed` - Stream Editor

```bash
sed 's/old/new/' file.txt        # Replace first occurrence per line
sed 's/old/new/g' file.txt       # Replace all occurrences
sed -i 's/old/new/g' file.txt    # Edit in place
sed -i.bak 's/old/new/g' file    # Edit with backup
sed 's/old/new/gi' file.txt      # Case-insensitive replace
sed '3s/old/new/' file.txt       # Replace only on line 3
sed '1,5s/old/new/g' file.txt    # Replace in lines 1-5
sed '/pattern/s/old/new/' file   # Replace only in matching lines
sed '/pattern/d' file.txt        # Delete lines matching pattern
sed '5d' file.txt                # Delete line 5
sed '1,5d' file.txt              # Delete lines 1-5
sed '/^$/d' file.txt             # Delete empty lines
sed 'G' file.txt                 # Double space file
sed -n '5p' file.txt             # Print only line 5
sed -n '5,10p' file.txt          # Print lines 5-10
sed -n '/pattern/p' file.txt     # Print matching lines only
sed '=' file.txt | sed 'N;s/\n/\t/'  # Add line numbers
```

## `awk` - Pattern Processing

```bash
awk '{print}' file.txt           # Print all lines
awk '{print $1}' file.txt        # Print first column
awk '{print $1, $3}' file.txt    # Print 1st and 3rd columns
awk '{print $NF}' file.txt       # Print last column
awk -F':' '{print $1}' /etc/passwd  # Custom delimiter
awk 'NR==5' file.txt             # Print line 5
awk 'NR>=5 && NR<=10' file.txt   # Print lines 5-10
awk '/pattern/' file.txt         # Print matching lines
awk '!/pattern/' file.txt        # Print non-matching lines
awk '{sum+=$1} END {print sum}'  # Sum first column
awk 'length > 80' file.txt       # Lines longer than 80 chars
awk '{print NR, $0}' file.txt    # Add line numbers
awk 'BEGIN {print "Header"} {print}' file  # Add header
awk '{gsub(/old/, "new"); print}' file     # Replace
awk -F',' '{print $1","$2}' file.csv       # CSV processing
awk '{for(i=1;i<=NF;i++) print $i}' file   # Print each word
```

## `cut` - Extract Columns

```bash
cut -c1-5 file.txt               # Characters 1-5
cut -c1,3,5 file.txt             # Characters 1, 3, and 5
cut -d',' -f1 file.csv           # First field (comma delimiter)
cut -d':' -f1,3 /etc/passwd      # Fields 1 and 3
cut -d',' -f2- file.csv          # From field 2 to end
cut -d',' --complement -f1 file  # All except field 1
```

## `sort` - Sort Lines

```bash
sort file.txt                    # Alphabetical sort
sort -r file.txt                 # Reverse sort
sort -n file.txt                 # Numeric sort
sort -h file.txt                 # Human numeric (1K, 2M)
sort -u file.txt                 # Remove duplicates
sort -t',' -k2 file.csv          # Sort by 2nd column
sort -t',' -k2 -n file.csv       # Numeric sort by 2nd column
sort -k2,2 -k1,1 file.txt        # Sort by col2, then col1
sort -f file.txt                 # Case-insensitive
sort -R file.txt                 # Random shuffle
sort -c file.txt                 # Check if sorted
```

## `uniq` - Remove Duplicates

```bash
uniq file.txt                    # Remove adjacent duplicates
sort file.txt | uniq             # Remove all duplicates
uniq -c file.txt                 # Count occurrences
uniq -d file.txt                 # Show only duplicates
uniq -u file.txt                 # Show only unique lines
uniq -i file.txt                 # Case-insensitive
```

## `wc` - Word Count

```bash
wc file.txt                      # Lines, words, characters
wc -l file.txt                   # Lines only
wc -w file.txt                   # Words only
wc -c file.txt                   # Bytes only
wc -m file.txt                   # Characters only
wc -L file.txt                   # Longest line length
wc -l *.txt                      # Count in multiple files
```

## `tr` - Translate Characters

```bash
tr 'a-z' 'A-Z' < file.txt        # Lowercase to uppercase
tr 'A-Z' 'a-z' < file.txt        # Uppercase to lowercase
tr -d '0-9' < file.txt           # Delete digits
tr -s ' ' < file.txt             # Squeeze repeated spaces
tr ':' '\t' < file.txt           # Replace : with tab
tr -d '\r' < file.txt            # Remove carriage returns
tr -cd 'a-zA-Z0-9' < file.txt    # Keep only alphanumeric
```

## `diff` - Compare Files

```bash
diff file1.txt file2.txt         # Show differences
diff -u file1.txt file2.txt      # Unified format
diff -y file1.txt file2.txt      # Side by side
diff -q file1.txt file2.txt      # Brief (just report if different)
diff -r dir1/ dir2/              # Compare directories
diff -i file1.txt file2.txt      # Ignore case
diff -w file1.txt file2.txt      # Ignore whitespace
diff -B file1.txt file2.txt      # Ignore blank lines
```

---

# 5. User Management

## User Information

```bash
whoami                           # Current username
id                               # User and group IDs
id username                      # Info for specific user
users                            # Logged in users
who                              # Who is logged in
w                                # Who is doing what
last                             # Login history
lastlog                          # Last login for all users
finger username                  # User information
```

## `useradd` / `adduser` - Create Users

```bash
sudo useradd username            # Create user (minimal)
sudo useradd -m username         # Create with home directory
sudo useradd -m -s /bin/bash user  # With home and shell
sudo useradd -m -g group user    # With primary group
sudo useradd -G sudo,docker user # With supplementary groups
sudo useradd -d /custom/home user  # Custom home directory
sudo useradd -e 2024-12-31 user  # Account expiry date
sudo useradd -c "Full Name" user # With comment/description

sudo adduser username            # Interactive user creation
```

## `usermod` - Modify Users

```bash
sudo usermod -l newname oldname  # Rename user
sudo usermod -d /new/home user   # Change home directory
sudo usermod -s /bin/zsh user    # Change shell
sudo usermod -aG sudo user       # Add to group (keep existing)
sudo usermod -G group1,group2 user  # Set groups (replace)
sudo usermod -L user             # Lock user account
sudo usermod -U user             # Unlock user account
sudo usermod -e 2024-12-31 user  # Set expiry date
```

## `userdel` - Delete Users

```bash
sudo userdel username            # Delete user
sudo userdel -r username         # Delete with home directory
sudo userdel -f username         # Force delete (even if logged in)
```

## `passwd` - Manage Passwords

```bash
passwd                           # Change own password
sudo passwd username             # Change user's password
sudo passwd -l username          # Lock account
sudo passwd -u username          # Unlock account
sudo passwd -e username          # Expire password (force change)
sudo passwd -d username          # Delete password (no password)
sudo passwd -S username          # Show password status
```

## Group Management

```bash
groups                           # Show current user's groups
groups username                  # Show user's groups
sudo groupadd groupname          # Create group
sudo groupdel groupname          # Delete group
sudo groupmod -n newname old     # Rename group
sudo gpasswd -a user group       # Add user to group
sudo gpasswd -d user group       # Remove user from group
newgrp groupname                 # Switch to group
```

## `sudo` - Superuser Do

```bash
sudo command                     # Run as root
sudo -u user command             # Run as specific user
sudo -i                          # Interactive root shell
sudo -s                          # Root shell (keep environment)
sudo -l                          # List allowed commands
sudo -k                          # Clear cached credentials
sudo !!                          # Run last command with sudo
sudo visudo                      # Edit sudoers file safely
```

## Important Files

```bash
cat /etc/passwd                  # User accounts
cat /etc/shadow                  # Password hashes (root only)
cat /etc/group                   # Groups
cat /etc/sudoers                 # Sudo permissions
```

---

# 6. Process Management

## `ps` - Process Status

```bash
ps                               # Current shell processes
ps aux                           # All processes (BSD style)
ps -ef                           # All processes (System V style)
ps -u username                   # User's processes
ps -p PID                        # Specific process
ps aux --sort=-%mem              # Sort by memory usage
ps aux --sort=-%cpu              # Sort by CPU usage
ps aux | grep nginx              # Find specific process
ps -eo pid,ppid,cmd,%mem,%cpu    # Custom output
ps --forest                      # Show process tree
ps -fp $(pgrep -d, -u user)      # User's processes detailed
```

## `top` & `htop` - Live Process Monitoring

```bash
top                              # Interactive process viewer
htop                             # Enhanced process viewer

# top shortcuts:
# q          - Quit
# k          - Kill process
# r          - Renice (change priority)
# M          - Sort by memory
# P          - Sort by CPU
# 1          - Show individual CPUs
# c          - Show full command
```

## `kill` - Terminate Processes

```bash
kill PID                         # Graceful termination (SIGTERM)
kill -9 PID                      # Force kill (SIGKILL)
kill -15 PID                     # Safe termination (SIGTERM)
kill -1 PID                      # Hangup (SIGHUP)
kill -STOP PID                   # Pause process
kill -CONT PID                   # Resume process
kill %1                          # Kill job 1
kill -l                          # List all signals
```

**Common Signals:**
| Signal | Number | Description |
|--------|--------|-------------|
| SIGHUP | 1 | Hangup / reload config |
| SIGINT | 2 | Interrupt (Ctrl+C) |
| SIGKILL | 9 | Force kill (can't be caught) |
| SIGTERM | 15 | Graceful termination |
| SIGSTOP | 17 | Pause process |
| SIGCONT | 19 | Resume process |

## `pkill` & `killall` - Kill by Name

```bash
pkill process_name               # Kill by name
pkill -9 process_name            # Force kill by name
pkill -u username                # Kill user's processes
pkill -f "pattern"               # Kill by full command line

killall process_name             # Kill all with name
killall -9 process_name          # Force kill all
killall -u username              # Kill user's processes
killall -i process_name          # Interactive
```

## `pgrep` - Find Process IDs

```bash
pgrep nginx                      # PIDs of nginx processes
pgrep -l nginx                   # PIDs with names
pgrep -u username                # User's PIDs
pgrep -f "pattern"               # Match full command
pgrep -c nginx                   # Count processes
pgrep -a nginx                   # PID and full command
```

## Background Jobs

```bash
command &                        # Run in background
jobs                             # List background jobs
jobs -l                          # List with PIDs
fg                               # Bring last job to foreground
fg %1                            # Bring job 1 to foreground
bg                               # Resume stopped job in background
bg %1                            # Resume job 1 in background
Ctrl+Z                           # Suspend current process
Ctrl+C                           # Terminate current process

nohup command &                  # Run immune to hangups
nohup command > out.log 2>&1 &   # With output redirection
disown %1                        # Remove job from shell
```

## `nice` & `renice` - Process Priority

```bash
nice -n 10 command               # Start with lower priority
nice -n -10 command              # Start with higher priority (root)
renice -n 10 -p PID              # Change priority
renice -n -5 -u username         # Change for user's processes
```

Priority range: -20 (highest) to 19 (lowest)

## Service Management (systemd)

```bash
# Service Control
sudo systemctl start nginx       # Start service
sudo systemctl stop nginx        # Stop service
sudo systemctl restart nginx     # Restart service
sudo systemctl reload nginx      # Reload config
sudo systemctl status nginx      # Check status
sudo systemctl enable nginx      # Start on boot
sudo systemctl disable nginx     # Don't start on boot
sudo systemctl is-active nginx   # Check if running
sudo systemctl is-enabled nginx  # Check if enabled

# System
sudo systemctl list-units        # List all units
sudo systemctl list-units --type=service  # List services
sudo systemctl --failed          # Show failed units
sudo systemctl daemon-reload     # Reload unit files

# Logs
journalctl -u nginx              # View service logs
journalctl -u nginx -f           # Follow logs
journalctl -u nginx --since "1 hour ago"
```

---

# 7. Disk & Storage

## Disk Usage

```bash
df                               # Disk space usage
df -h                            # Human-readable
df -hT                           # With filesystem type
df -i                            # Inode usage
df /path                         # Specific mount point

du                               # Directory space usage
du -h                            # Human-readable
du -sh *                         # Summary of each item
du -sh directory/                # Total size of directory
du -h --max-depth=1              # One level deep
du -h --max-depth=2 | sort -h    # Two levels, sorted
du -ch *.log                     # Total of .log files
du -a | sort -n | tail -10       # Largest 10 files
```

## `lsblk` - List Block Devices

```bash
lsblk                            # List block devices
lsblk -f                         # With filesystem info
lsblk -a                         # All devices
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT  # Custom columns
```

## `fdisk` - Partition Management

```bash
sudo fdisk -l                    # List all partitions
sudo fdisk -l /dev/sda           # List specific disk
sudo fdisk /dev/sda              # Partition disk (interactive)
```

## `mount` & `umount` - Mount Filesystems

```bash
mount                            # Show mounted filesystems
mount | column -t                # Formatted output
sudo mount /dev/sdb1 /mnt        # Mount partition
sudo mount -t ext4 /dev/sdb1 /mnt  # Specify type
sudo mount -o ro /dev/sdb1 /mnt  # Mount read-only
sudo mount -a                    # Mount all in fstab
sudo umount /mnt                 # Unmount
sudo umount -l /mnt              # Lazy unmount
cat /etc/fstab                   # Permanent mounts
```

## `mkfs` - Create Filesystem

```bash
sudo mkfs.ext4 /dev/sdb1         # Create ext4 filesystem
sudo mkfs.xfs /dev/sdb1          # Create XFS filesystem
sudo mkfs.vfat /dev/sdb1         # Create FAT32 filesystem
```

## `dd` - Disk Duplication

```bash
# Create bootable USB
sudo dd if=image.iso of=/dev/sdb bs=4M status=progress

# Backup disk
sudo dd if=/dev/sda of=/backup/disk.img bs=4M

# Wipe disk with zeros
sudo dd if=/dev/zero of=/dev/sdb bs=4M
```

> ⚠️ **WARNING**: `dd` can destroy data. Triple-check your `of=` target!

---

# 8. LVM & Partition Management

## Understanding Storage Hierarchy

```
Physical Disks (/dev/sda, /dev/sdb)
        │
        ▼
Partitions (/dev/sda1, /dev/sdb1)
        │
        ▼
Physical Volumes (PV)
        │
        ▼
Volume Groups (VG)
        │
        ▼
Logical Volumes (LV)
        │
        ▼
Filesystems (ext4, xfs)
        │
        ▼
Mount Points (/home, /var)
```

## Partition Management

### `fdisk` - MBR Partition Tool (< 2TB disks)

```bash
# List all partitions
sudo fdisk -l
sudo fdisk -l /dev/sda           # Specific disk

# Interactive partition management
sudo fdisk /dev/sdb

# fdisk interactive commands:
# m     - Help menu
# p     - Print partition table
# n     - New partition
# d     - Delete partition
# t     - Change partition type
# w     - Write changes and exit
# q     - Quit without saving
# g     - Create new GPT partition table
# o     - Create new MBR partition table
```

**Example: Create a new partition**
```bash
sudo fdisk /dev/sdb
# Command: n          (new partition)
# Partition type: p   (primary)
# Partition number: 1
# First sector: Enter (default)
# Last sector: +10G   (10GB size)
# Command: t          (change type)
# Hex code: 8e        (Linux LVM)
# Command: w          (write and exit)
```

### `parted` - GPT Partition Tool (> 2TB disks)

```bash
# List partitions
sudo parted -l
sudo parted /dev/sdb print       # Specific disk

# Interactive mode
sudo parted /dev/sdb

# Create GPT partition table
sudo parted /dev/sdb mklabel gpt

# Create partition (non-interactive)
sudo parted /dev/sdb mkpart primary ext4 0% 50%
sudo parted /dev/sdb mkpart primary ext4 50% 100%

# parted interactive commands:
# help      - Show commands
# print     - Show partitions
# mklabel   - Create partition table (gpt/msdos)
# mkpart    - Create partition
# rm        - Remove partition
# resizepart - Resize partition
# name      - Name partition
# quit      - Exit

# Resize partition
sudo parted /dev/sdb resizepart 1 20GB

# Set partition flags
sudo parted /dev/sdb set 1 lvm on
sudo parted /dev/sdb set 1 boot on
```

### `gdisk` - GPT fdisk

```bash
sudo gdisk /dev/sdb              # Interactive GPT partitioning
sudo gdisk -l /dev/sdb           # List GPT partitions

# gdisk commands similar to fdisk:
# n - New partition
# d - Delete partition
# p - Print table
# w - Write and exit
# t - Change type (8e00 for LVM)
```

### Partition Types Reference

| Code | Type | Description |
|------|------|-------------|
| 83 | Linux | Standard Linux partition |
| 82 | Linux swap | Swap partition |
| 8e | Linux LVM | LVM physical volume |
| fd | Linux raid | RAID partition |
| 07 | NTFS | Windows NTFS |
| 0c | FAT32 | Windows FAT32 |

---

## LVM - Logical Volume Management

### Why Use LVM?
- ✅ **Flexible resizing** - Grow/shrink volumes on the fly
- ✅ **Snapshots** - Point-in-time copies for backups
- ✅ **Spanning** - Combine multiple disks into one volume
- ✅ **Striping** - Improve performance across disks
- ✅ **Easy migration** - Move data between physical disks

### LVM Components

```
┌─────────────────────────────────────────────────────────────┐
│                    LOGICAL VOLUMES (LV)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │  lv_root │  │  lv_home │  │  lv_var  │                   │
│  │   20GB   │  │   50GB   │  │   30GB   │                   │
│  └──────────┘  └──────────┘  └──────────┘                   │
├─────────────────────────────────────────────────────────────┤
│                    VOLUME GROUP (VG)                         │
│                      vg_data (100GB)                         │
├─────────────────────────────────────────────────────────────┤
│                  PHYSICAL VOLUMES (PV)                       │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │   /dev/sdb1      │  │   /dev/sdc1      │                 │
│  │      50GB        │  │      50GB        │                 │
│  └──────────────────┘  └──────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

### Physical Volumes (PV)

```bash
# Create PV from partition
sudo pvcreate /dev/sdb1
sudo pvcreate /dev/sdc1

# Create PV from whole disk (not recommended)
sudo pvcreate /dev/sdd

# Create multiple PVs at once
sudo pvcreate /dev/sdb1 /dev/sdc1 /dev/sdd1

# Display PV information
sudo pvs                         # Brief summary
sudo pvdisplay                   # Detailed info
sudo pvdisplay /dev/sdb1         # Specific PV

# Scan for PVs
sudo pvscan

# Remove PV
sudo pvremove /dev/sdb1

# Move data off a PV (before removal)
sudo pvmove /dev/sdb1

# Move to specific PV
sudo pvmove /dev/sdb1 /dev/sdc1

# Resize PV (after partition resize)
sudo pvresize /dev/sdb1
sudo pvresize --setphysicalvolumesize 40G /dev/sdb1
```

### Volume Groups (VG)

```bash
# Create VG from PVs
sudo vgcreate vg_data /dev/sdb1 /dev/sdc1

# Create with specific extent size (default 4MB)
sudo vgcreate -s 16M vg_data /dev/sdb1

# Display VG information
sudo vgs                         # Brief summary
sudo vgdisplay                   # Detailed info
sudo vgdisplay vg_data           # Specific VG

# Scan for VGs
sudo vgscan

# Extend VG (add more PVs)
sudo vgextend vg_data /dev/sdd1

# Reduce VG (remove PV - must move data first)
sudo pvmove /dev/sdb1            # Move data off
sudo vgreduce vg_data /dev/sdb1  # Remove from VG

# Rename VG
sudo vgrename vg_data vg_storage

# Remove VG (must remove LVs first)
sudo vgremove vg_data

# Activate/Deactivate VG
sudo vgchange -ay vg_data        # Activate
sudo vgchange -an vg_data        # Deactivate

# Export/Import VG (for moving to another system)
sudo vgexport vg_data            # Export
sudo vgimport vg_data            # Import
```

### Logical Volumes (LV)

```bash
# Create LV with size
sudo lvcreate -L 10G -n lv_data vg_data

# Create LV with percentage of VG
sudo lvcreate -l 50%VG -n lv_data vg_data

# Create LV using all free space
sudo lvcreate -l 100%FREE -n lv_data vg_data

# Create with specific number of extents
sudo lvcreate -l 100 -n lv_data vg_data

# Display LV information
sudo lvs                         # Brief summary
sudo lvdisplay                   # Detailed info
sudo lvdisplay /dev/vg_data/lv_data  # Specific LV

# Scan for LVs
sudo lvscan

# Extend LV
sudo lvextend -L +5G /dev/vg_data/lv_data     # Add 5GB
sudo lvextend -L 20G /dev/vg_data/lv_data     # Extend to 20GB
sudo lvextend -l +100%FREE /dev/vg_data/lv_data  # Use all free space

# Extend LV and resize filesystem together
sudo lvextend -r -L +5G /dev/vg_data/lv_data

# Reduce LV (DANGEROUS - backup first!)
sudo umount /dev/vg_data/lv_data
sudo e2fsck -f /dev/vg_data/lv_data   # Check filesystem
sudo resize2fs /dev/vg_data/lv_data 8G # Shrink filesystem first
sudo lvreduce -L 8G /dev/vg_data/lv_data

# Rename LV
sudo lvrename vg_data lv_data lv_storage

# Remove LV
sudo umount /dev/vg_data/lv_data
sudo lvremove /dev/vg_data/lv_data

# Activate/Deactivate LV
sudo lvchange -ay /dev/vg_data/lv_data  # Activate
sudo lvchange -an /dev/vg_data/lv_data  # Deactivate
```

### LVM Snapshots

```bash
# Create snapshot (requires free space in VG)
sudo lvcreate -L 5G -s -n lv_data_snap /dev/vg_data/lv_data

# Create thin snapshot (more efficient)
sudo lvcreate -s -n lv_data_snap /dev/vg_data/lv_data

# Mount snapshot (read-only recommended)
sudo mount -o ro /dev/vg_data/lv_data_snap /mnt/snapshot

# Restore from snapshot (revert to snapshot state)
sudo umount /dev/vg_data/lv_data
sudo lvconvert --merge /dev/vg_data/lv_data_snap
sudo mount /dev/vg_data/lv_data /mnt/data

# Remove snapshot
sudo lvremove /dev/vg_data/lv_data_snap

# Check snapshot usage
sudo lvs -o +snap_percent
```

### Thin Provisioning

```bash
# Create thin pool
sudo lvcreate -L 100G --thinpool thin_pool vg_data

# Create thin volume (can exceed pool size - overprovisioning)
sudo lvcreate -V 200G --thin -n thin_vol vg_data/thin_pool

# Monitor thin pool usage
sudo lvs -o +data_percent,metadata_percent

# Extend thin pool
sudo lvextend -L +50G vg_data/thin_pool
```

---

## Complete LVM Workflow Example

### Scenario: Add new 100GB disk and create LVM

```bash
# 1. Identify new disk
lsblk
# Output shows /dev/sdb as new 100GB disk

# 2. Create partition for LVM
sudo fdisk /dev/sdb
# n -> p -> 1 -> Enter -> Enter -> t -> 8e -> w

# 3. Create Physical Volume
sudo pvcreate /dev/sdb1
sudo pvs  # Verify

# 4. Create Volume Group
sudo vgcreate vg_app /dev/sdb1
sudo vgs  # Verify

# 5. Create Logical Volumes
sudo lvcreate -L 30G -n lv_www vg_app
sudo lvcreate -L 20G -n lv_logs vg_app
sudo lvcreate -l 100%FREE -n lv_data vg_app
sudo lvs  # Verify

# 6. Create Filesystems
sudo mkfs.ext4 /dev/vg_app/lv_www
sudo mkfs.xfs /dev/vg_app/lv_logs
sudo mkfs.ext4 /dev/vg_app/lv_data

# 7. Create Mount Points
sudo mkdir -p /var/www /var/log/app /data

# 8. Mount Volumes
sudo mount /dev/vg_app/lv_www /var/www
sudo mount /dev/vg_app/lv_logs /var/log/app
sudo mount /dev/vg_app/lv_data /data

# 9. Add to /etc/fstab for persistence
echo '/dev/vg_app/lv_www  /var/www      ext4 defaults 0 2' | sudo tee -a /etc/fstab
echo '/dev/vg_app/lv_logs /var/log/app  xfs  defaults 0 2' | sudo tee -a /etc/fstab
echo '/dev/vg_app/lv_data /data         ext4 defaults 0 2' | sudo tee -a /etc/fstab

# 10. Verify
df -h
mount | grep vg_app
```

### Expanding LVM When Disk Runs Out

```bash
# Scenario: /var/www is running out of space

# Option 1: If VG has free space
sudo vgs  # Check VG free space
sudo lvextend -r -L +10G /dev/vg_app/lv_www

# Option 2: Add new disk to VG
# Add new disk /dev/sdc
sudo pvcreate /dev/sdc1
sudo vgextend vg_app /dev/sdc1
sudo lvextend -r -L +50G /dev/vg_app/lv_www
```

---

## Filesystem Operations on Volumes

### Create Filesystems

```bash
sudo mkfs.ext4 /dev/vg_data/lv_data    # ext4
sudo mkfs.xfs /dev/vg_data/lv_data     # XFS
sudo mkfs.btrfs /dev/vg_data/lv_data   # Btrfs

# With options
sudo mkfs.ext4 -L "DataVolume" /dev/vg_data/lv_data  # With label
sudo mkfs.xfs -f /dev/vg_data/lv_data   # Force (overwrite)
```

### Resize Filesystems

```bash
# ext4 - Can grow online, shrink offline
sudo resize2fs /dev/vg_data/lv_data           # Grow to fill LV
sudo resize2fs /dev/vg_data/lv_data 10G       # Shrink to 10G (offline)

# XFS - Can only grow, not shrink
sudo xfs_growfs /mount/point                   # Grow to fill LV
sudo xfs_growfs -D 10485760 /mount/point      # Grow to specific blocks

# Combined LV extend + filesystem resize
sudo lvextend -r -L +10G /dev/vg_data/lv_data
```

### Check & Repair Filesystems

```bash
# ext4
sudo e2fsck -f /dev/vg_data/lv_data      # Force check
sudo e2fsck -p /dev/vg_data/lv_data      # Auto-repair

# XFS
sudo xfs_repair /dev/vg_data/lv_data
sudo xfs_repair -n /dev/vg_data/lv_data  # Dry run
```

---

## Swap Management

### Create Swap Partition

```bash
# Create swap partition with fdisk (type 82)
sudo fdisk /dev/sdb
# n -> p -> 2 -> Enter -> +4G -> t -> 2 -> 82 -> w

# Format as swap
sudo mkswap /dev/sdb2

# Enable swap
sudo swapon /dev/sdb2

# Add to fstab
echo '/dev/sdb2 swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

### Create Swap File

```bash
# Create swap file
sudo fallocate -l 4G /swapfile
# Or: sudo dd if=/dev/zero of=/swapfile bs=1M count=4096

# Set permissions
sudo chmod 600 /swapfile

# Format as swap
sudo mkswap /swapfile

# Enable swap
sudo swapon /swapfile

# Add to fstab
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab

# Verify
swapon --show
free -h
```

### Create LVM Swap Volume

```bash
sudo lvcreate -L 4G -n lv_swap vg_data
sudo mkswap /dev/vg_data/lv_swap
sudo swapon /dev/vg_data/lv_swap
echo '/dev/vg_data/lv_swap swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

### Manage Swap

```bash
swapon --show                    # Show active swap
sudo swapon -a                   # Enable all swap in fstab
sudo swapoff /dev/sdb2           # Disable specific swap
sudo swapoff -a                  # Disable all swap
cat /proc/swaps                  # Swap details
```

---

## Useful LVM Commands Summary

| Command | Description |
|---------|-------------|
| `pvs` / `pvdisplay` | Physical Volume info |
| `vgs` / `vgdisplay` | Volume Group info |
| `lvs` / `lvdisplay` | Logical Volume info |
| `pvcreate` | Initialize PV |
| `pvremove` | Remove PV |
| `pvmove` | Move data between PVs |
| `vgcreate` | Create VG |
| `vgextend` | Add PV to VG |
| `vgreduce` | Remove PV from VG |
| `lvcreate` | Create LV |
| `lvextend` | Grow LV |
| `lvreduce` | Shrink LV |
| `lvremove` | Delete LV |
| `lvrename` | Rename LV |

---

# 9. Networking

## Network Configuration

```bash
ip addr                          # Show IP addresses
ip addr show eth0                # Specific interface
ip link                          # Show network interfaces
ip link set eth0 up              # Enable interface
ip link set eth0 down            # Disable interface
ip route                         # Show routing table
ip route add default via 192.168.1.1  # Add default gateway
ip neigh                         # ARP table

ifconfig                         # Legacy: show interfaces
ifconfig eth0                    # Legacy: specific interface
ifconfig eth0 192.168.1.10       # Legacy: set IP
```

## Connectivity Testing

```bash
ping google.com                  # ICMP ping
ping -c 5 google.com             # 5 pings only
ping -i 0.5 google.com           # 0.5 second interval
ping -s 1000 google.com          # Custom packet size

traceroute google.com            # Trace packet route
traceroute -n google.com         # Without DNS resolution
mtr google.com                   # Combined ping + traceroute

curl https://api.example.com     # HTTP request
curl -I example.com              # Headers only
curl -X POST -d "data" url       # POST request
curl -H "Auth: token" url        # Custom header
curl -o file.zip url             # Download to file
curl -O url                      # Download (keep filename)
curl -L url                      # Follow redirects
curl -k url                      # Ignore SSL errors

wget https://example.com/file    # Download file
wget -O newname.zip url          # Custom filename
wget -c url                      # Resume download
wget -r -np url                  # Recursive download
wget -q url                      # Quiet mode
```

## DNS Tools

```bash
nslookup google.com              # DNS lookup
nslookup -type=MX google.com     # MX records
dig google.com                   # DNS query
dig google.com A                 # A records
dig google.com MX                # MX records
dig +short google.com            # Brief output
dig @8.8.8.8 google.com          # Use specific DNS server
host google.com                  # DNS lookup (simple)
```

## Port & Connection Analysis

```bash
netstat -tuln                    # Listening ports
netstat -tulnp                   # With process info
netstat -an                      # All connections
netstat -rn                      # Routing table

ss -tuln                         # Modern netstat
ss -tulnp                        # With processes
ss -s                            # Socket statistics
ss -t state established          # Established connections

lsof -i                          # Internet connections
lsof -i :80                      # Processes on port 80
lsof -i -P -n                    # All with ports
```

## Network File Transfer

```bash
# SCP - Secure Copy
scp file.txt user@host:/path/    # Copy to remote
scp user@host:/path/file.txt .   # Copy from remote
scp -r dir/ user@host:/path/     # Copy directory
scp -P 2222 file user@host:/     # Custom port

# RSYNC - Advanced sync
rsync -av source/ dest/          # Archive + verbose
rsync -avz source/ user@host:/dest/  # With compression
rsync -avz --delete src/ dest/   # Sync and delete extras
rsync -avz --exclude='*.log' src/ dest/  # Exclude pattern
rsync -avz --progress src/ dest/ # Show progress
rsync -avzn src/ dest/           # Dry run
```

## SSH - Secure Shell

```bash
ssh user@host                    # Connect
ssh -p 2222 user@host            # Custom port
ssh -i key.pem user@host         # With key file
ssh -L 8080:localhost:80 user@host  # Local port forward
ssh -R 8080:localhost:80 user@host  # Remote port forward
ssh -D 8080 user@host            # SOCKS proxy
ssh -N -f -L 8080:localhost:80 user@host  # Background tunnel

# SSH Keys
ssh-keygen -t rsa -b 4096        # Generate RSA key
ssh-keygen -t ed25519            # Generate ED25519 key
ssh-copy-id user@host            # Copy key to host
cat ~/.ssh/id_rsa.pub            # View public key
```

## Firewall (UFW & iptables)

```bash
# UFW - Uncomplicated Firewall
sudo ufw status                  # Check status
sudo ufw enable                  # Enable firewall
sudo ufw disable                 # Disable firewall
sudo ufw allow 22                # Allow SSH
sudo ufw allow 80/tcp            # Allow HTTP
sudo ufw deny 23                 # Deny port 23
sudo ufw allow from 192.168.1.0/24  # Allow subnet
sudo ufw delete allow 80         # Delete rule
sudo ufw reset                   # Reset all rules

# iptables
sudo iptables -L                 # List rules
sudo iptables -L -v -n           # Verbose + numeric
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -s 10.0.0.0/8 -j DROP
sudo iptables -F                 # Flush all rules
sudo iptables-save > rules.txt   # Save rules
sudo iptables-restore < rules.txt  # Restore rules
```

---

# 9. Package Management

## APT (Debian/Ubuntu)

```bash
sudo apt update                  # Update package list
sudo apt upgrade                 # Upgrade packages
sudo apt full-upgrade            # Complete upgrade
sudo apt install package         # Install package
sudo apt remove package          # Remove package
sudo apt purge package           # Remove + config files
sudo apt autoremove              # Remove unused deps
sudo apt search keyword          # Search packages
sudo apt show package            # Package info
apt list --installed             # List installed
apt list --upgradable            # List upgradable
sudo apt-get clean               # Clear package cache
sudo apt-mark hold package       # Prevent upgrades
```

## YUM/DNF (RHEL/CentOS/Fedora)

```bash
sudo dnf update                  # Update packages
sudo dnf install package         # Install package
sudo dnf remove package          # Remove package
sudo dnf search keyword          # Search packages
sudo dnf info package            # Package info
sudo dnf list installed          # List installed
sudo dnf clean all               # Clear cache
sudo dnf history                 # Transaction history
sudo dnf history undo <id>       # Undo transaction

# YUM (older systems)
sudo yum update
sudo yum install package
sudo yum remove package
```

## Snap & Flatpak

```bash
# Snap
snap find keyword                # Search snaps
sudo snap install package        # Install
sudo snap remove package         # Remove
snap list                        # List installed
sudo snap refresh package        # Update package
sudo snap refresh                # Update all

# Flatpak
flatpak search keyword           # Search
flatpak install flathub app.id   # Install
flatpak uninstall app.id         # Remove
flatpak list                     # List installed
flatpak update                   # Update all
```

---

# 10. System Monitoring

## System Information

```bash
uname -a                         # All system info
uname -r                         # Kernel version
uname -m                         # Architecture
hostnamectl                      # Hostname + OS info
cat /etc/os-release              # OS details
cat /proc/version                # Kernel version
lsb_release -a                   # Distribution info
uptime                           # System uptime
date                             # Current date/time
cal                              # Calendar
```

## Hardware Information

```bash
lscpu                            # CPU info
lsmem                            # Memory summary
free -h                          # Memory usage
cat /proc/meminfo                # Detailed memory
lspci                            # PCI devices
lsusb                            # USB devices
lshw                             # All hardware
sudo dmidecode                   # BIOS/hardware
cat /proc/cpuinfo                # CPU details
```

## System Logs

```bash
# Journal (systemd)
journalctl                       # All logs
journalctl -b                    # Current boot
journalctl -b -1                 # Previous boot
journalctl --since "1 hour ago"  # Time filter
journalctl -p err                # Errors only
journalctl -f                    # Follow logs
journalctl _SYSTEMD_UNIT=nginx   # Specific unit

# Traditional logs
tail -f /var/log/syslog          # System log
tail -f /var/log/auth.log        # Auth log
tail -f /var/log/nginx/access.log  # App logs
cat /var/log/dmesg               # Kernel ring buffer
dmesg                            # Kernel messages
dmesg -T                         # Human-readable time
```

## Performance Monitoring

```bash
vmstat 1                         # Virtual memory stats
iostat -x 1                      # I/O statistics
mpstat 1                         # CPU stats per processor
sar -u 1 5                       # CPU usage
sar -r 1 5                       # Memory usage
sar -b 1 5                       # I/O usage
iotop                            # I/O by process
iftop                            # Network by connection
nload                            # Network load
```

---

# 11. Compression & Archives

## `tar` - Tape Archive

```bash
# Create archives
tar -cvf archive.tar files/      # Create tar
tar -czvf archive.tar.gz files/  # Create tar.gz
tar -cjvf archive.tar.bz2 files/ # Create tar.bz2
tar -cJvf archive.tar.xz files/  # Create tar.xz

# Extract archives
tar -xvf archive.tar             # Extract tar
tar -xzvf archive.tar.gz         # Extract tar.gz
tar -xjvf archive.tar.bz2        # Extract tar.bz2
tar -xJvf archive.tar.xz         # Extract tar.xz
tar -xzvf archive.tar.gz -C /dir/  # Extract to directory

# List contents
tar -tvf archive.tar             # List tar contents
tar -tzvf archive.tar.gz         # List tar.gz contents

# Options:
# c - create    x - extract    t - list
# v - verbose   f - file       z - gzip
# j - bzip2     J - xz
```

## `gzip`, `bzip2`, `xz` - Compression

```bash
gzip file.txt                    # Compress (replaces original)
gzip -k file.txt                 # Keep original
gzip -d file.txt.gz              # Decompress
gunzip file.txt.gz               # Decompress (same as -d)
gzip -l file.txt.gz              # Show compression info

bzip2 file.txt                   # Better compression
bzip2 -k file.txt                # Keep original
bunzip2 file.txt.bz2             # Decompress

xz file.txt                      # Best compression
xz -k file.txt                   # Keep original
unxz file.txt.xz                 # Decompress
```

## `zip` & `unzip`

```bash
zip archive.zip file1 file2      # Create zip
zip -r archive.zip directory/    # Zip directory
zip -e archive.zip files         # Encrypted zip
unzip archive.zip                # Extract
unzip archive.zip -d /path/      # Extract to directory
unzip -l archive.zip             # List contents
unzip -o archive.zip             # Overwrite without prompt
```

---

# 12. Shell Scripting Essentials

## Variables

```bash
# Define variables (no spaces around =)
NAME="John"
AGE=25
FILES=$(ls)
TODAY=$(date +%Y-%m-%d)

# Use variables
echo $NAME
echo "Hello, ${NAME}!"
echo "Age: $AGE"

# Special variables
$0                # Script name
$1, $2, ...       # Positional arguments
$#                # Number of arguments
$@                # All arguments as separate strings
$*                # All arguments as one string
$?                # Last command exit status
$$                # Current process ID
$!                # Last background process ID
```

## Conditionals

```bash
# If statement
if [ condition ]; then
    commands
elif [ condition ]; then
    commands
else
    commands
fi

# File tests
[ -f file ]       # File exists
[ -d dir ]        # Directory exists
[ -e path ]       # Path exists
[ -r file ]       # Readable
[ -w file ]       # Writable
[ -x file ]       # Executable
[ -s file ]       # Not empty

# String tests
[ -z "$str" ]     # Empty string
[ -n "$str" ]     # Not empty
[ "$a" = "$b" ]   # Equal
[ "$a" != "$b" ]  # Not equal

# Numeric tests
[ $a -eq $b ]     # Equal
[ $a -ne $b ]     # Not equal
[ $a -lt $b ]     # Less than
[ $a -le $b ]     # Less or equal
[ $a -gt $b ]     # Greater than
[ $a -ge $b ]     # Greater or equal

# Logical operators
[ cond1 ] && [ cond2 ]    # AND
[ cond1 ] || [ cond2 ]    # OR
[ ! condition ]            # NOT
```

## Loops

```bash
# For loop
for i in 1 2 3 4 5; do
    echo $i
done

for file in *.txt; do
    echo "Processing $file"
done

for ((i=0; i<10; i++)); do
    echo $i
done

# While loop
while [ condition ]; do
    commands
done

# Until loop
until [ condition ]; do
    commands
done

# Read file line by line
while IFS= read -r line; do
    echo "$line"
done < file.txt
```

## Functions

```bash
# Define function
my_function() {
    echo "Arguments: $@"
    echo "First arg: $1"
    local var="local variable"
    return 0
}

# Call function
my_function arg1 arg2
result=$?
```

## I/O Redirection

```bash
command > file           # Redirect stdout to file
command >> file          # Append stdout to file
command 2> file          # Redirect stderr to file
command 2>&1             # Redirect stderr to stdout
command &> file          # Redirect both to file
command < file           # Read input from file
command1 | command2      # Pipe stdout to next command
command << EOF           # Here document
text
EOF
```

## Useful Script Patterns

```bash
#!/bin/bash
set -e              # Exit on error
set -u              # Exit on undefined variable
set -o pipefail     # Exit on pipe failure

# Error handling
trap 'echo "Error on line $LINENO"' ERR

# Check root
if [ "$EUID" -ne 0 ]; then
    echo "Please run as root"
    exit 1
fi

# Default values
NAME=${1:-"default"}

# Check command exists
if ! command -v docker &> /dev/null; then
    echo "Docker not found"
    exit 1
fi
```

---

# 13. Advanced Commands

## `xargs` - Build Commands from Input

```bash
find . -name "*.txt" | xargs rm          # Delete found files
find . -name "*.log" | xargs -I {} rm {} # Placeholder
echo "a b c" | xargs -n 1                # One per line
cat list.txt | xargs -P 4 command        # Parallel execution
ls | xargs -I {} mv {} {}.bak            # Rename all files
```

## `tee` - Write to File and stdout

```bash
command | tee file.txt           # Output to both
command | tee -a file.txt        # Append
command 2>&1 | tee log.txt       # Include stderr
```

## `screen` & `tmux` - Terminal Multiplexers

```bash
# Screen
screen                           # New session
screen -S name                   # Named session
screen -ls                       # List sessions
screen -r name                   # Reattach
Ctrl+a d                         # Detach

# Tmux
tmux                             # New session
tmux new -s name                 # Named session
tmux ls                          # List sessions
tmux attach -t name              # Attach
Ctrl+b d                         # Detach
Ctrl+b %                         # Split vertical
Ctrl+b "                         # Split horizontal
Ctrl+b arrow                     # Switch pane
```

## `crontab` - Scheduled Tasks

```bash
crontab -e                       # Edit crontab
crontab -l                       # List crontab
crontab -r                       # Remove crontab

# Cron format:
# * * * * * command
# │ │ │ │ │
# │ │ │ │ └─ Day of week (0-7, 0/7=Sunday)
# │ │ │ └─── Month (1-12)
# │ │ └───── Day of month (1-31)
# │ └─────── Hour (0-23)
# └───────── Minute (0-59)

# Examples:
0 5 * * * /scripts/backup.sh     # Daily at 5 AM
*/15 * * * * /scripts/check.sh   # Every 15 minutes
0 0 * * 0 /scripts/weekly.sh     # Weekly on Sunday
0 */2 * * * command              # Every 2 hours
```

## `watch` - Repeat Commands

```bash
watch df -h                      # Monitor disk space
watch -n 5 "ps aux | head"       # Every 5 seconds
watch -d ls -la                  # Highlight changes
watch -g command                 # Exit on change
```

## `strace` & `ltrace` - Debugging

```bash
strace command                   # Trace system calls
strace -p PID                    # Trace running process
strace -e open command           # Filter by call type
strace -o output.log command     # Log to file
ltrace command                   # Trace library calls
```

## `lsof` - List Open Files

```bash
lsof                             # All open files
lsof -u username                 # User's open files
lsof +D /path/                   # Files in directory
lsof -i :80                      # Processes on port 80
lsof -c nginx                    # Files opened by nginx
lsof -p PID                      # Files by process
lsof /path/to/file               # Who's using file
```

## Environment Variables

```bash
env                              # Show all
printenv                         # Show all
printenv PATH                    # Specific variable
echo $PATH                       # Print variable
export VAR="value"               # Set for session
export PATH="$PATH:/new/path"    # Add to PATH
unset VAR                        # Remove variable

# Permanent: add to ~/.bashrc or ~/.profile
echo 'export VAR="value"' >> ~/.bashrc
source ~/.bashrc                 # Reload
```

## System Control

```bash
sudo shutdown now                # Shutdown immediately
sudo shutdown -r now             # Reboot immediately
sudo shutdown -h +10             # Shutdown in 10 minutes
sudo shutdown -c                 # Cancel shutdown
sudo reboot                      # Reboot
sudo poweroff                    # Power off
sudo halt                        # Halt system
```

---

# Quick Reference Cheat Sheet

## Most Used Commands
| Command | Description |
|---------|-------------|
| `cd` | Change directory |
| `ls -la` | List all files with details |
| `pwd` | Print working directory |
| `cp -r` | Copy recursively |
| `mv` | Move/rename |
| `rm -rf` | Force remove recursively |
| `mkdir -p` | Create nested directories |
| `touch` | Create file |
| `cat` | Display file |
| `grep -r` | Search recursively |
| `find . -name` | Find files |
| `chmod` | Change permissions |
| `chown` | Change ownership |
| `ps aux` | All processes |
| `kill -9` | Force kill |
| `df -h` | Disk space |
| `du -sh` | Directory size |
| `top/htop` | Process monitor |
| `ssh` | Remote connection |
| `scp/rsync` | File transfer |

## Keyboard Shortcuts
| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Kill current process |
| `Ctrl+Z` | Suspend process |
| `Ctrl+D` | Exit / EOF |
| `Ctrl+L` | Clear screen |
| `Ctrl+A` | Go to line start |
| `Ctrl+E` | Go to line end |
| `Ctrl+R` | Search history |
| `Ctrl+U` | Delete to start |
| `Ctrl+K` | Delete to end |
| `Tab` | Auto-complete |
| `↑/↓` | History navigation |

---

> 📝 **Note**: Practice these commands regularly in a safe environment. Create test directories and files to experiment with before working on production systems.

> ⚠️ **Warning**: Always be extra careful with commands like `rm -rf`, `dd`, and anything run with `sudo`. Double-check your commands before executing!

---

*Last Updated: January 2026*
*Created for DevOps Engineers*
