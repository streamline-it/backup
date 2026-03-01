Role Name
=========

Performs backups.  This is capable of backing up entire directories using rsync and individual file(s) using the copy command.

This will configure the backup host which uses nfs to share it's backup locations.  It will also configure the backup remote servers.  The "backup_operation" variable will control what will get configured.  If this variable is set to "h" it will configure the backup host.  If the variable is set to "r", it configures the remote host.  The "remote host" in this case is the host that has files to be backed up to the backup host.

This role allows you to backup to an an "external" drive which can be any type of storage device.  And it allows you to set a "cloud-based" backup option.

Requirements
------------

Will run on RHEL 7 and later and FreeBSD systems

Role Variables
--------------

# As a backup host this will have the capability of backing up multiple hosts.  The backup_remote_hostname variable will
# be used to create a directory, all backups for that host will exist under it.  The backup host will also have 2 locatiions
# for backups.  One on the live filesystem, the second on an optional external drive, and the 3rd optional backup location
# can be a remote cloud backup.  Variables will control whether the external drive (backup_external_drive_option: true/false) or
# the cloud backup (backup_cloud_backup_option: true/false) will be used.
# The host will have a script that will drive where to save the backups, the backup host's only responsibility is to take that  
# latest backup and push it to the external and/or cloud backup.

# for backup_operation if = r - Run for the remote host
#                      if = h - Run this for the backup host itself
#

backup_operation: r or h

# directory where backups are stored

backup_directory:
  dir_name: /exp/nfs/priv/backups
  owner: root
  groups: root

# location specs
backup_logfile_path: "{{ backup_directory['dir_name'] }}/backup_log.txt"
backup_script_directory: /root/scripts/backup
backup_archive_directory: /exp/nfs/priv/archive
# this is used for the external drive backup.  Backups
# in this file are used as a queue of what needs to be copied to 
# the external drive
backup_queue_filepath: "{{ backup_directory['dir_name'] }}/queue.txt"

# purpose is to place the backups underneath a directory of the host's name
backup_remote_hostname: host

# remote host's mount point for backup host
backup_mount_point: /backup

# if true the backup host will duplicate the latest backup to an external drive
backup_external_drive_option: false

# external drive device; used to create the partition and filesystem
# added fullname to adjust for FreeBSD's partitions  ex device is /dev/da0, the entire device name including the
# partition would be /dev/da0p1.  Also added a label which I prefer in FreeBSD

backup_external_drive:
  - dev_name: /dev/<device>
    size: 1G
    fstype: xfs
    mount_fstype: xfs
    mount_point: /mnt/backup
    full_name: /dev/<device><partition>
    label: <label>

# If true the latest backup will duplicated to a remote cloud provider from the backup host
backup_cloud_backup_option: false

# Below are the cronjob variables a cronjob for the backup host specific to the remote drive option
backup_external_drive_kickoff_time_hour: 11
backup_external_drive_kickoff_time_minute: 0
backup_external_drive_kickoff_time_day: '*'
backup_external_drive_kickoff_time_month: '*'
backup_external_drive_kickoff_time_weekday: '0-7'
backup_external_drive_cronjob_name: ext_drive_backup
backup_external_drive_cronjob_job: "{{  backup_script_directory }}/{{ backup_external_drive_scripts[1]['script_name']}}"


# Below are the cronjob variables a cronjob for the backup host specific to the cloud option
backup_cloud_backup_kickoff_time_hour: 13
backup_cloud_backup_kickoff_time_minute: 0
backup_cloud backup_kickoff_time_day: '*'
backup_cloud backup_kickoff_time_month: '*'
backup_cloud backup_kickoff_time_weekday: '0-7'
backup_cloud_backup_cronjob_name: cloud_backup
backup_cloud_backup_cronjob_job: "echo 'setup your cronjob for cloud backup'"

# method to mount the external drive
# values: device: mount using the device name
#         label: mount via the label
backup_mount_method: device

# user to run the cronjob for
backup_cronjob_user: root

# for backup_locations_to_backup, this variable will be used with the backup script in a rsync command.  The "exclude_filter" will 
# include the directories/files to include.  The hostname variable will be the hostname of the computer that is being backed up.
# the backup_cmd dictionary will contain the backup command (either rsync or the cloud-specific command).  The directory holds the name of the
# directory the files will exist under.  For example if backing up /home/rich, the directory will hold home/rich.  Remember that the
# host_name value will be the parent directory for all backups, for that host.
# Basically this backup_locations_to_backup dictionary list will be used to create individual rsync or remote commands for the backup
# script to be run on the remote host.

#Note on attribute (type).  Values are "full" only backup when specifying a full backup or "partial" for abbreviated backups
 
backup_locations_to_backup:
  - source_path: /home
    backup_type: <full/partial>
    file_path: ""
    remote_path: /backup
    source_mounted: true
    exclude_filter: ''
    host_name: host
    backup_cmd: rsync -a
    directory: <directory>
    file_copy: false
    enabled: true

# disk label (can be  msdos/gpt) used for FreeBSD hosts
backup_freebsd_disk_scheme: gpt


# This is needed for FreeBSD systems with the preflight script which uses the du and an exclude option to exlude certain directories.  FreeBSD's du doesn't include this option.

# backup_excude_directories:
#  - path: ./priv/videos
#  - path: ./priv/pics

backup_exclude_directories: []

vars/main.yml

# vars file for backup

vars/FreeBSD_script_vars.yml

# bacckup scripts
backup_scripts:
  - template_name: BackupData-bourne.sh.j2
    script_name: BackupData.sh
  - template_name: preflight_remote-bourne.sh.j2
    script_name: preflight_remote.sh

backup_external_drive_scripts:
  - template_name: BackupData-external_drive_-bourne.sh.j2
    script_name: BackupData-external_drive.sh
  - template_name: preflight_external_drive-bourne.sh.j2
    script_name: preflight_external_drive-remote.sh

backup_root_group: wheel

backup_misc_pkgs:
  - rsync

vars/Redhat_script_vars.yml

# backup scripts
backup_scripts:
  - template_name: BackupData-bash.sh.j2
    script_name: BackupData.sh
  - template_name: preflight_remote-bash-shell.sh.j2
    script_name: preflight_remote.sh

backup_external_drive_scripts:
  - template_name: BackupData-external_drive-bash.sh.j2
    script_name: BackupData.sh
  - template_name: preflight_external_drive-bash-shell.sh.j2
    script_name: preflight_external_drive.sh

backup_root_group: root

backup_misc_pkgs:
  - rsync

Dependencies
------------

None.

Example Playbook
----------------

I prefer setting the variables in the var host files.  Easier maintenence.  

Backup host setup:

- hosts: backup_server

  vars:

    backup_operation: h
    backup_external_drive_option: true
    backup_cloud_backup_option: false


    backup_directory:
      dir_name: /exp/nfs/priv/backups
      owner: root
      groups: nfsusers

    backup_external_drive:
      - dev_name: /dev/vdb
        size: 1G
        fstype: xfs
        mount_fstype: xfs
        mount_point: /mnt/extdrive
        full_name: /dev/vdb1
        label: backups

    backup_mount_point: /mnt/extdrive
    backup_external_drive_kickoff_time_hour: 12
    backup_external_drive_kickoff_time_minute: 25
    backup_external_drive_kickoff_time_day: '*'
    backup_external_drive_kickoff_time_month: '*'
    backup_external_drive_kickoff_time_weekday: '0-7'
    backup_external_drive_cronjob_name: ext_drive_backup
    backup_external_drive_cronjob_job: "{{ backup_script_directory}}/{{ backup_external_drive_scripts[1]['script_name'] }}"

  roles:

        - name: backup


Setup for the remote host:

- hosts: clients

  vars:

    my_backup_server_ip: 172.16.20.87

    backup_operation: r
    backup_remote_hostname: 172.16.20.85

    backup_directory:
      dir_name: /exp/nfs/priv/backups
      owner: root
      groups: nfsusers

    my_backup_server_ip: 172.16.20.87

    backup_remote_hostname: 172.16.20.85
    backup_external_drive_option: true
    backup_mount_method: device
    # remote host's cronjob to back itself up
    backup_remote_host_backup_kickoff_time_hour: 11
    backup_remote_host_backup_kickoff_time_minute: 33
    backup_remote_host_backup_kickoff_time_day: '*'
    backup_remote_host_backup_kickoff_time_month: '*'
    backup_remote_host_backup_kickoff_time_weekday: '0-7'
    backup_remote_host_backup_cronjob_name: host_backup
    backup_remote_host_backup_cronjob_job: "{{ backup_script_directory}}/{{ backup_scripts[1]['script_name'] }}"

    backup_locations_to_backup:
      - source_path: /home
        backup_type: full
        file_path: ""
        remote_path: "{{ my_backup_server_ip }}:{{ backup_directory['dir_name'] }}"
        source_mounted: true
        exclude_filter: ''
        host_name: 172.16.20.85
        backup_cmd: rsync -a
        directory: "{{ backup_directory['dir_name'] }}"
        file_copy: false
        enabled: true

  roles:

    - name: backup

License
-------

BSD

Author Information
------------------

Rich O'Hara

