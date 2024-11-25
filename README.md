# criecm.zfs

Setup ZFS on system

Create zfs filesystems / volumes
  * integrates with `criecm.samba`, `criecm.nfs_server` and `criecm.ctld`
  * replication (every hour by default)
  * restauration available ;)

## Requirements

Filer with zfs !

## Role Variables

  * `shares`: list of `share` dicts
  * `myzfs ('')` override to process only this share.name
  * `zfs_snap_default_retention ('')` activate automatic snapshots - see `snap_retention`

### share variables
  * `zfsrc` (MANDATORY)
    ZFS source
  * `path` (MANDATORY for zfs filesystem)
    if defined, will set the mountpoint of a filesystem
  * `volsize` (MANDATORY for zfs volume if no origin)
    if defined, will create a volume instead of filesystem
  * `zfsprops` ({})
    dict of zfs properties to set
  * `mode ()`
    if you need to chmod new dir instead of default umask
  * `owner ()`
    Change from default "root" owner
  * `group ()`
    Change from default group (0)
  * `acls ([])`
    If you need to apply acls (only on create)
  * `nfsv4_acls (True)`
    acls are NFSv4, not POSIX

#### if you want automated snapshots, add:
  * `snap_retention (zfs_snap_default_retention)`
    `''` to disable snapshots for this share
    expression of what to keep: 6l12h7d3w3m1y means:
      6 last snaps
      12 last hourly snaps
      7 last daily snaps
      3 last weekly snaps
      3 last monthly snaps
      1 last yearly
  * `snap_minute (0)`
    at which minute(s) snapshots are taken - crontab syntax
      `0`: each hour at h00
      `5`: each hour at h05
      `*/5`: each 5 minutes
  * `snap_hour (*)`
    at which hour (all by default) - crontab syntax

#### if you want PRA replication (same filesystems, snapshots and properties on destination), add:
  * `pra_on ('')`
    "host:zfs/path" to set up PRA replication
  * `pra_minute ({pseudo-random})`
     minute field for crontab
     if you replicate many filesystems more than once per hour, take care of
     choosing different minutes to avoid launching all at the same time
  * `pra_hour (*)`
     hour field for crontab

#### if you want replication, add:
  * `sync_on ([])`
     list of machine:zfs/path to replicate this zfs to
  * `sync_minute ({pseudo-random})`
     minute field for crontab
     if you replicate many filesystems more than once per hour, take care of
     choosing different minutes to avoid launching all at the same time
  * `sync_hour (*)`
     hour field for crontab

#### if you want old-school replication, add:
  * `backup_on` ('')
    "zfs/path@host" to set up synchronization
  * `backup_minute` ({pseudo-random})
    minute field for crontab
    if you replicate many filesystems more than once per hour, take care of
    choosing different minutes to avoid launching all at the same time
  * `backup_hour` (`*`)
    hour field for crontab

## replication - global vars
  * [`zfs_sync_vol`](https://github.com/criecm/savscript/raw/master/lib/zfs_sync_vol) script for replication
    if not installed, the role will download it to /root/zfs_sync_vol on backup machine(s)
    (the role may be easily adapted to use another tool)
  * [`zfs_pra_scripts`](https://forge.centrale-marseille.fr/projects/sysutils/repository/zfs/revisions/master/show/pra) directory containing scripts for `zfs send | receive` over ssh, must pre-exist on source and destination
  * [`zfs_sync_scripts`](https://forge.centrale-marseille.fr/projects/sysutils/repository/zfs/revisions/master/show/sync) directory containing scripts for `zfs send | receive` over ssh using bookmarks on source
  * [`zfs_snap_script`](https://github.com/criecm/zfstools) path to zfs_snap_make script (it needs zfs_clean_snap too)

## restauration
  - delete any share mountpoint to restore them
  - add `-e restore=1` to import from backup host
  - play again without `restore` to install sync

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: server
      roles:
        - criecm.zfs
      vars:
        shares:
          # simple zfs filesystem
          - zfsrc: zdata/my/files
            path: /shares/myfiles

          # simple zfs volume
          - zfsrc: zdata/volz/myvol1
            volsize: '25G'

          # more complex filesystem
          - zfsrc: zdata/my/filesystems
            zfsprops:
              atime: 'off'
              refquota: '5G'
              aclmode: passthrough
            owner: memyselfandi
            mode: '0700'

          # replicated volumes container
          - zfsrc: zdata/rvolz
            path: 'none'
            # replicate to otherhost
            backup_on: zdata/rvolz@otherhost
            # backup every 4 hour
            backup_hour: '*/4'

          # this volume will be replicated via above container
          - name: simple-repl-vol
            zfsrc: zdata/rvolz/mysecondvol
            volsize: '15G'
    
          # cifs (with shadow copy) & nfs shared, acl enabled, replicated fs
          # (you'll only need to list criecm.samba and/or criecm.nfs_server here, as
          #  both will `include_role: criecm.zfs`)
          - name: sales
            zfsrc: zdata/depts/sales
            zfsprops:
              aclmode: passthrough
              aclinherit: passthrough
            group: salers
            mode: '0770'
            acls:
              - 'group:salctrls:full_set:fd'
              - 'group:salctrls:read_set:fd'
              - 'group:auditors:read_set:fd'
            cifs: Yes
            smbparams:
              comment: 'Y: on shaun'
              write list: '@admin_dsi @prof'
              read only: 'No'
              vfs objects: zfsacl shadow_copy2
              nfs4:chown: 'yes'
              nfs4:acedup: 'merge'
              nfs4:mode: 'special'
              shadow:sort: desc
              shadow:format: 'GMT-%Y.%m.%d-%H.%M.%S'
              shadow:snapdir: .zfs/snapshot
              inherit owner: 'Yes'
              inherit permissions: 'Yes'
            nfsshares:
              - "-maproot=0 -network=192.0.2.0/24"
              - "-network=203.0.113.0/24"

## License

BSD

## Author Information

https://dsi.centrale-marseille.fr/
