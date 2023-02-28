---
title: Backup local Data to external drive bash script
date: 2023-01-17 5:00:00 
categories: [linux, script]
tags: [linux, ubuntu, backup, script]
layout: post
toc: true
comments: false
math: false
mermaid: false
---

I wanted to automate local machine data backup to external drive. First I was checking is there somekind of backup solutions already but then I thought it might be better to practice my bash scripting skills which are not good say the least. 

### Logic
1. Mount external drive
2. Copy every specified folder to external drive as TAR file
3. cleanup "old" TAR-files
4. unmount external drive 

>**The Importance of Unmounting**  
>Failure to unmount before disconnecting the device can result in loss of data and/or a corrupted file system.

Here's a walkthrough of my creation. Full script can be found [here](https://github.com/apaivinen/scripts/blob/main/Bash/Backup%20data%20to%20external%20drive/backup.sh)

## Functions & configs
### Functions
First part of the script contains functions which I needed
```bash
# Function, Define a timestamp
function timestamp() {
          local NOW=$(date +"%Y-%m-%d-%H-%M-%S") # current time
          echo $NOW
}

# Function, create filename
function createFileName(){
          local parameter=$1
          local now=$(timestamp)
          local fileName="$parameter-$now"
          echo $fileName
}
```

With these functions I can create file name with time stamps. The format I wanted to have is `name-YYYY-MM-DD-HH-MM-SS`

`timestamp()` doesn't take any parameters, it just creates current time as `YYYY-MM-DD-HH-MM-SS`

`createFileName()` takes one parameter and sets it to `parameter`-variable. This is due to my lack of bash understanding and `$1` just seems to be too confusing currently.
Function gets timestamp from `timestamp`-function to `now`-variable.
Both variables are combined to `fileName`-variable.
createFile-function is called from main section when creating backup file name. 

I know I could have combined those two functions in to one but *it is what it is*.

### Configs

```bash
# CONFIGS
FoldersToBackup=(
"/PathToFolder/folder1"
"/PathToFolder/folder2"
"/PathToFolder/folder3"
)

# Get device path by using sudo fdisk -l
Device="/dev/sdb1"
# Destination where device is mounted and files are copied
Destination="/media/external-backup"
```

There's simple array with full paths of folders which needs to be backed up.
I was thinking maybe I could create object array which contains folder path and name for each folder. With this way I could give a custom backup name if I wanted to. But then I realized this is too advanced for me right now. Let's keep it simple for now. 

There's also basic variables for mounting/unmounting the drive.
> **IMPORTANT** 
> create /media/external-backup location before running the script.

```important
sudo mkdir /media/external-backup
```

## Main script
### Mount backup location as active
```bash
echo "Mounting drive "$Device "to" $Destination
sudo mount -t ntfs-3g $Device $Destination
```

I'm using external hard drive which is formated to ntfs due to my need to use it on windows machines without any problems.

### Create backups

```bash
echo "Starting backup operation"
for sourcePath in ${FoldersToBackup[@]}; do
          folderName=$(basename $sourcePath)
          FileName=$(createFileName "$folderName")
          destinationPath=$Destination"/"$FileName".tar.gz"
          sudo tar -zcvf $destinationPath $sourcePath
          echo $destinationPath
done
```

Just a simple for-loop to read `FoldersToBackup`-array.
I'm exctracting folder name by using basename to `folderName`-variable.
`FileName` variable is getting return value from `createFileName`-function to which I pass `folderName`-variable. `FileName` should be something like `folder1-2023-02-12-18-01-15` 
`destinationPath` variable is created by using `Destination` and `FileName` variables. 
then compress source folder to destination location with tar command
z = Use gzip
c = Create  a  new  archive.
v = List the name of each file as it is processed
f = Next argument is file name of the archive (or destination path)

Also for debugging this command list contents of .tar.gz file `tar -tvf /media/external-backup/folder1-2023-02-12-18-01-15` 
t = List files
v = List the name of each file as it is processed
f = Next argument is file name of the archive (or full path)

### Cleanup backups
```bash
#Delete old files
echo Cleaning up old backups
find $Destination -name "*.tar.gz" -type f -mtime +60 -delete
ls -lah $Destination
```
Using find command to find all files ending with `.tar.gz` from destination location, filter out all recent backups and deleting old backups.  
name = every file ending .tar.gz
type f = Restrict find to just files
mtime +60 = Filter out everything newer than 60 days ago modified files
delete = delete results which are returned from previous filtering

### Unmount drive
```bash
echo "Unmounting" $Destination
sudo umount $Device
```
Nothing fancy just unmounting operation

## docs
Usefull information about mounting: https://kenfavors.com/code/mounting-an-external-drive-on-ubuntu-server/