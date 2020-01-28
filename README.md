# Overview

Importing content in a disconnected environment can be a challenge.
These scripts make use of the Inter-Satellite Sync capability in Satellite 6 to
allow for full and incremental export/import of content between environments.

These scripts have been written and tested using Satellite 6.x on RHEL7. (RHEL6 not supported)
Export/Import testing has been performed on the following version combinations:

* 6.2 -> 6.2
* 6.2 -> 6.3
* 6.3 -> 6.3
* 6.3 -> 6.2
* 6.3 -> 6.4
* 6.4 -> 6.4
* 6.4 -> 6.3

## Definitions

Throughout these scripts the following references are used:

* Connected Satellite: Internet connection is available
* Disconnected Satellite: No internet connection is available
* Sync Host: Connected Satellite that downloads and exports content for a Disconnected Satellite

## Requirements

* Satellite >= 6.2.9
* Python = 2.7
* PyYAML (python2-pyyaml)
* Simplejson (python2-simplejson)
* requests (python2-requests)

The Export and Import scripts are intended to be run on the Satellite servers directly.

* sat_export is intended to run on the Connected Satellite,
* sat_import is intended to run on the Disconnected Satellite.
* The scripts make use of the Satellite REST API, and require an admin account on the Satellite server.

## Install Instructions

1.  Create a service account for API use in each targeted Satellite server.

    ```bash
    $ hammer user create --login svc-sat-api-user --firstname Satellite \
      --lastname APIUser --password='<PASSWORD>' --mail no-reply@example.org \
      --auth-source-id 1 --organization-ids 1 --default-organization-id 1 \
      --admin true
    ```

2.  Install the sat6_scripts RPM, available from
    [the GitHub Releases page](https://github.com/RedHatSatellite/sat6_scripts/releases).
3.  Configure the `/usr/share/sat6_scripts/config.yml` and
    `/usr/share/sat6_scripts/exports.yml` config files as detailed in the
    [Configuration section](#configuration). If copying files in from somewhere
    else, make sure to set the permissions appropriately (0600).

### Set up exporting Satellite server

1.  Foreman needs to be configured to export content to the location you require. By default the path is
    /var/lib/pulp/katello-export - this will result in you probably filling your /var/lib/pulp partition!
    The configs in these scripts assume that the exports are going to /var/sat-export - this should be a
    dedicated partition or even better dedicated disk just for export content.
    1.  (Recommended) Add a logical volume and mount it to `/var/sat-export`. The size of this logical
        volume should be double the size of the existing disk for `/var/lib/pulp` to allow
        for full exports to exist at the same time as a previous full export.
2.  Configure Satellite to export to the new volume:

    ```bash
    $ hammer settings set --name pulp_export_destination --value /var/sat-export
    $ chown foreman:foreman /var/sat-export
    $ semanage fcontext -a -t foreman_var_run_t '/var/sat-export(/.*)?'
    $ restorecon -RvF /var/sat-export
    ```

3.  Set the default download policy.

    ```bash
    $ hammer settings set --name default_download_policy --value immediate
    ```

    If this is being run after Satellite is already set up, each existing
    repository will need its Download Policy set to Immediate. See the solution
    guide for instructions to set these:
    [How to change download policy of repositories in Red Hat Satellite 6?](https://access.redhat.com/solutions/3481621)

### Set up importing Satellite server

1.  Add a logical volume and mount it to `/var/sat-content`. The size of this
    logical volume should be approximately the same size of the connected
    Satellite’s disk for /var/lib/pulp, or less if this Satellite won't be an
    exact mirror.

## Assumptions

For content import to a disconnected Satellite, it is assumed that the relevant
subscription manifest has been copied to and uploaded in the disconnected satellite.
For the _import with sync_ option (default), required Red Hat repositories on the
disconnected satellite must have already been enabled, and any custom repositories created.

For custom repos, the names of the product and the repositories within __MUST__ match on
both the connected and disconnected Satellites.

## Configuration

A YAML based configuration file is in config/config.yml.example  
The example file needs to be copied to config/config.yml and customised as
required:

```yaml
satellite:
  url: https://sat6.example.org
  username: svc-api-user
  password: 1t$a$3cr3t
  disconnected: [True|False]     (Is direct internet connection available?)
  manifest: my-satellite         (Red Hat Portal satellite application name)
  default_org: MyOrg             (Default org to use - can be overridden with -o)

logging:
  dir: /var/log/sat6-scripts     (Directory to use for logging)
  debug: [True|False]

email:
  mailout: True
  mailfrom: Satellite 6 <sat62@example.org>
  mailto:
    - sysadmin@example.org

export:
  dir: /var/sat-export           (Directory to export content to - Connected Satellite)

import:
  dir: /var/sat-content          (Directory to import content from - Disconnected Satellite)
  syncbatch: 50                  (Number of repositories to sync at once during import)
```


## Log files

The scripts in this project will write output to satellite.log in the directory
specified in the config file.

## Scripts in this project

### check_sync

A quick method to check the status of sync tasks from the command line.
Will show any sync tasks that have stuck in a 'paused' state, as well as any
tasks that have stopped but been marked as Incomplete.
Running with the -l flag will loop the check until terminated with CTRL-C

### download_manifest

Script originally written by Rich Jerrido downloads subscription manifest from
Red Hat portal. Need to provide a username (-l) with access to download the
manifest. Manifest name is configured in config.yml, but can be overridden on the
command line with the (-s) option. Manifest is downloaded to the the configured
export directory, and included in the bundle generated by sat_export.

### sat_export

Intended to perform content export from a Connected Satellite (Sync Host), for
transfer into a disconnected environment. The Default Organization View (DOV)
is exported by default, meaning that there is no specific requirement to create
lifecycle environments or Content Views on the sync host. If there is a requirement
to export only certain repositories, this can also be specified using additional
configuration files. The following export types can be performed:

* A full export (-a)
* An incremental export of content since the last successful export (-i)
* An incremental export of content from a given date (-s)
* Export of a limited repository set (-e) defined by config file (see below)

By default, the exported RPMs are verified for GPG integrity before being
added to a chunked tar archive, with each part of the archive being sha256sum'd
for cross domain transfer integrity checking.

The GPG check requires that GPG keys are imported into the local RPM GPG store.
The RPM GPG keys must be installed on the connected satellite.

```bash
rpm --import <gpg-key>
```

If there is a need to NOT perform the GPG check of the exported packages, the
GPG check can be skipped using the (-n) option.

For each export performed, a log of all RPM packages that are exported is kept
in the configured log directory. This has been found to be a useful tool to see
when (or if) a specific package has been imported into the disconnected host.

Satellite will export the repodata of all included repositories, even if the
repo has no content to export. The way the sat_import script works is that it will
perform a sync on these 'empty' repos, consuming time and resources, especially if
multiple capsules are then synced as well. By default, 'empty' repos are not included
for import sync, however this behaviour can be overridden with the (-r) flag. This
will be useful to periodically ensure that the disconnected satellite repos are
consistent - the repodata will indicate mismatches with synced content.

The exported content will be archived in TAR format, with a chunk size specified
by the (-S) option. The default is 4200Mb.

To export a selected repository set, the exports.yml config file must exist in the
config directory. The format of this file is shown below, and contains one or more
'env' stanzas, containing a list of repositories to export. The repository name is
the LABEL taken from the Satellite server.

Note also that the 'environment' export option also allows for the export of ISO
(file) based repositories in addition to yum RPM content. Using the DOV export does
NOT include ISO or Puppet repos, as export of these type of repositories is not
supported by the current pulp version in Satellite. The 'environment' export performs
some additional magic to export the file and puppet content.

```yaml
exports:
  env1:
    name: DEVELOPMENT
    repos:
      - Red_Hat_Satellite_6_2_for_RHEL_7_Server_RPMs_x86_64
      - Red_Hat_Satellite_Tools_6_2_for_RHEL_7_Server_RPMs_x86_64
      - Red_Hat_Enterprise_Linux_7_Server_Kickstart_x86_64_7_3
      - Red_Hat_Enterprise_Linux_7_Server_RPMs_x86_64_7Server
      - Red_Hat_Enterprise_Linux_7_Server_-_Extras_RPMs_x86_64
      - Red_Hat_Enterprise_Linux_7_Server_-_Optional_RPMs_x86_64_7Server
      - Red_Hat_Enterprise_Linux_7_Server_-_RH_Common_RPMs_x86_64_7Server
      - Red_Hat_Software_Collections_RPMs_for_Red_Hat_Enterprise_Linux_7_Server_x86_64_7Server
      - Red_Hat_Enterprise_Linux_7_Server_ISOs_x86_64_7Server
      - epel-7-x86_64
      - Puppet_Forge

  env2:
    name: TEST
    repos:
      - Red_Hat_Satellite_Tools_6_2_for_RHEL_6_Server_RPMs_x86_64
      - Red_Hat_Satellite_6_2_for_RHEL_7_Server_RPMs_x86_64
```

To export in this manner the '-e DEVELOPMENT' option must be used.
Exports to the 'environment' will be timestamped in the same way that DOV exports
are done, so ongoing incremental exports are possible.

In the event that a Puppet repository is exported, it will be done such that the
connected satellite can import that repository. In some situations, an offline
Puppet Forge mirror (puppet-forge-server ruby gem) is used to facilitate r10k use
of Puppet Forge modules. This is not part of the Satellite infrastructure, however the
puppet module export can be performed so that it is consumable by puppet-forge-server
as well as Satellite - this is done using the -p flag, and results in a /puppetforge
directory being written to the import directory during the sat_import process.

#### sat_export Help Output

```bash
usage: sat_export.py [-h] [-o ORG] [-e ENV] [-a | -i | -s SINCE] [-l] [-n] [-S SIZE]

Performs Export of Default Content View.

optional arguments:
  -h, --help            show this help message and exit
  -o ORG, --org ORG     Organization (Uses default if not specified)
  -e ENV, --env ENV     Environment config
  -a, --all             Export ALL content
  -i, --incr            Incremental Export of content since last run
  -s SINCE, --since SINCE
                        Export content since YYYY-MM-DD HH:MM:SS
  -l, --last            Display time of last export
  -L, --list            List all successfully completed exports
  --nogpg               Skip GPG checking
  -u, --unattended      Answer any prompts safely, allowing automated usage
  -r, --repodata        Include repodata for repos with no incremental content
  -S, --splitsize       Size of split files in Megabytes, defaults to 4200
  -p, --puppetforge     Include puppet-forge-server format Puppet Forge repo
  --notar               Do not archive the extracted content
  --forcexport          Force export from an import-only (Disconnected) Satellite
```

#### sat_export Examples

```bash
./sat_export.py -e DEV              # Incr export of repos defined in the DEV config
./sat_export.py -o AnotherOrg       # Incr export of DoV for a different org
./sat_export.py -e DEV -a           # Full export of repos defined in the DEV config

Output file format will be:
sat_export_20160729-1021_DEV_00
sat_export_20160729-1021_DEV_01
sat_export_20160729-1021_DEV.sha256
```

### sat_import

This companion script to sat_export, running on the Disconnected Satellite
performs a sha256sum verification of each part of the specified archive prior
to extracting the transferred content to disk.

Once the content has been extracted, a check is performed to see if any exports
performed have not yet been imported. This is to assist with data integrity on
the disconnected Satellite system. Any missing imports will be displayed and the
option to continue or abort will be presented. Upon continuing, a sync is triggered
of each repository in the import set. Note that repositories MUST be enabled on the
disconnected satellite prior to the sync working - for this reason a `nosync`
option (-n) exists so that the repos can be extracted to disk and then enabled
before the sync occurs. In order to not overload the Satellite during the sync, the
repositories will be synced in smaller batches, the number of repos in a batch
being defined in the config.yml file. (It has been observed on systems with a
large number of repos that triggering a sync on all repos at once pretty much
kills the Satellite until the sync is complete)

All imports are treated as Incremental, and the source tree will be removed on
successful import/sync.

The input archive files can also be automatically removed on successful import/sync
with the (-r) flag.

The last successfully completed import can be identified with the (-l) flag.
All previously imported datasets can be shown with the (-L) flag.

Note that a dataset can normally only be imported ONCE. To force an import of an
already completed dataset, use the (-f) flag.

In the event that missing import datasets are detected, they should be imported to
ensure data integrity and consistency. There may however be cases that result in
the missing imports being included by other means, or no longer required at all.
In these cases, the --fixhistory flag can be used to 'reset' the import history
so that it matches the export history of the current import dataset, clearing
these warnings.

#### sat_import Help Output

```bash
usage: sat_import.py [-h] [-o ORG] -d DATE [-n] [-r] [-l] [-L] [-c] [-f] [--fixhistory] [-u]

Performs Import of Default Content View.

optional arguments:
  -h, --help            show this help message and exit
  -o ORG, --org ORG     Organization (Uses default if not specified)
  -d DATE, --date DATE  Date/name of Import fileset to process (YYYY-MM-DD_NAME)
  -n, --nosync          Do not trigger a sync after extracting content
  -r, --remove          Remove input files after import has completed
  -l, --last            Show the last successfully completed import date
  -L, --list            List all successfully completed imports
  -c, --count           Display all package counts after import
  -f, --force           Force import of data if it has previously been done  
  -u, --unattended      Answer any prompts safely, allowing automated usage
  --fixhistory          Force import history to match export history  
```

#### sat_import Examples

```bash
./sat_import.py -d 20160729-1021_DEV -n         # Import content defined in DEV.yml but do not sync
./sat_import.py -d 20160729-1021_DoV            # Extract a DoV export but do not sync it
./sat_import.py -o MyOrg -l                     # Lists the date of the last successful import
./sat_import.py -o AnotherOrg -d 20160729-1021_DEV # Import content for a different org
```

### push_puppetforge

This script allows users with an offline puppet-forge-server (rubygem) instance to
perform a special export of puppetforge modules from the Satellite puppet-forge
repository (-r) in the directory structure required by the puppet-forge-server
application. After exporting, the modules are copied to the puppet-forge-server. The format
of the export is controlled with the type (-t) flag, as either 'puppet-forge-server' for the
rubygem based server, or 'artifactory' for JFrog Artifactory puppet server format.
The puppet-forge-server hostname can be defined in the config.yml, or overridden with
(-s), as can the module path (-m) on the remote server (default is /opt/puppet-forge/modules).
The user performing the rsync will be the user that is running the script, unless
overridden with (-u).

The config.yml block that defines the puppet-forge-server hostname is:

```yaml
puppet-forge-server:
  servertype: puppet-forge-server
  hostname: puppetforge.example.org
  modulepath: /opt/puppet-forge/modules
  username: someuser
  token: ArtifactoryAPIToken
```

#### push_puppetforge Help Output

```bash
usage: push_puppetforge.py [-h] [-o ORG] [-r REPO] [-t TYPE] [-s SERVER] [-m MODULEPATH] [-u USER]

Exports puppet modules in puppet-forge-server format.

optional arguments:
  -h, --help            show this help message and exit
  -o ORG, --org ORG     Organization (Uses default if not specified)
  -r REPO, --repo REPO  Puppetforge repository label
  -t TYPE, --type TYPE  Puppetforge server type (puppet-forge-server|artifactory)
  -s SERVER, --server SERVER
                        puppet-forge-server hostname
  -m MODULEPATH, --modulepath MODULEPATH
                        path to puppet-forge-server modules
  -u USER, --user USER  Username to push modules to server as (default is user
                        running script)
  -p PASSWORD --password PASSWORD
                        Token for Artifactory API authentication
```

#### push_puppetforge Examples

```bash
./push_puppetforge.py -r Puppet_Forge  
./push_puppetforge.py -r Puppet_Forge -u fred  
./push_puppetforge.py -r Puppet_Forge -s test.example.org -m /opt/tmp -t artifactory
```

### clean_content_views

This script removes orphaned versions of either all or nominated content views.
This should be run periodically to clean out old/unused content view data from
the mongo database and improve the responsiveness of the Satellite server.
Any orphaned versions older than the last in-use version are purged (orphans
between in-use versions will NOT be purged, unless the cleanall (-c) option is used).
There is a keep (-k) option that allows a specific number of versions older than the
last in-use to be kept as well, allowing for possible rollback of versions.
Note that the (-c) option will delete ALL orphaned versions, regardless of the (-k)
value. An alternative option (-i) slightly alters the calculation of the versions
that will be deleted, in that the (-k) option defines how many versions after the
newest (last promoted) version will be kept, rather than use the oldest (first
promoted) version.

Content views to clean can be defined by either:

* Specific content views defined in the main config file
* All content views (-a)

The option to use will depend on the historic (old) content views you wish to keep.
An example of the different options with a keep value of '1' is shown below:

```sql
+-----------------+----------+-----------------------+------------+
| version         | no flags | --ignorefirstpromoted | --cleanall |
+-----------------+----------+-----------------------+------------+
| 110.0 (Library) |          |                       |            |
| 109.0           |   KEEP   |         KEEP          |    DEL     |
| 108.3           |   KEEP   |         DEL           |    DEL     |
| 108.2 (Quality) |          |                       |            |
| 108.1           |   KEEP   |         DEL           |    DEL     |
| 108.0           |   DEL    |         DEL           |    DEL     |
| 107.0           |   DEL    |         DEL           |    DEL     |
+-----------------+----------+-----------------------+------------+
```

The dry run (-d) option can be used to see what would be published for a
given command input. Use this option to see the difference in behaviour between
(-a) and (-i) options, with and without (-c)

The defaults are configured in the main config.yml file in a YAML block like this:

```yaml
cleanup:
  content_views:
    - view: RHEL Server
      keep: 1
    - view: RHEL Workstation
      keep: 3
```

This configuration will clean only the two listed content views, and keep the
specified number of versions beyond the oldest in-use.

#### clean_content_views Help Output

```bash
usage: clean_content_views.py [-h] [-o ORG] [-a] [-c] [-d] [-i]

Cleans content views for specified organization.

optional arguments:
  -h, --help            show this help message and exit
  -o ORG, --org ORG     Organization (Uses default if not specified)
  -k KEEP, --keep KEEP  How many old versions to keep (only used with -a)
  -a, --all             Clean ALL content views
  -c, --cleanall        Remove orphan versions between in-use views
  -i, --ignorefirstpromoted  Version to keep count starts from first CV, not first promoted CV
  -d, --dryrun          Dry Run - Only show what will be cleaned
```

#### clean_content_views Examples

```bash
./clean_content_views.py          # Clean views using the default config
./clean_content_views.py -a -k 2  # Clean all views but keep the 2 oldest
./clean_content_views.py -a -d    # Show what would be done to clean all views
```

### publish_content_views

Publishes new content to the Library environment. The following can be published:

* Specific content views defined in the main config file
* All content views (-a)

The dry run (-d) option can be used to see what would be published for a
given command input. By default progress bars will be displayed for each content
view being published. It may be desirable to not display these (e.g. automating), so
they can be disabled using the (-q) option.

A default comment will be added to the published content view version containing the
username of the user that initiated the publish. Alternatively, a custom comment
can be added to the published view using the (-c) option.

Each time a content view is published or promoted, a datestamp is recorded so that
the last publish/promote date can be viewed with the (-l) option. Note that this
datestamp is only updated by this script - it does NOT record publish/promote via
the WebUI or Hammer CLI.

By default the repository metadata is not rebuilt. In some scenarios it may be required
to force a rebuild of the metadata, in which case the (-m) option can be used to trigger this.

The defaults are configured in the main config.yml file in a YAML block like this:

```yaml
publish:
  batch: 10
  content_views:
    - RHEL Server
    - RHEL Workstation
```

This configuration will publish only the two listed content views.

The batch: parameter can be used to limit the number of content views that will be published at
once, to aid in performance tuning.

#### publish_content_views Help Output

```bash
usage: publish_content_views.py [-h] [-o ORG] [-a] [-d] [-c COMMENT] [-m] [-q] [-l]

Publishes content views for specified organization.

optional arguments:
  -h, --help         show this help message and exit
  -o ORG, --org ORG  Organization (Uses default if not specified)
  -a, --all          Publish ALL content views
  -d, --dryrun       Dry Run - Only show what will be published
  -l, --last         Display last promotions
  -c, --comment      Add a custom description
  -q, --quiet        Suppress progress output updates
  -m, --forcemeta    Force metadata regeneration
```

#### publish_content_views Examples

```bash
./publish_content_views.py                   # Publish the default content views
./publish_content_views.py -a                # Publish ALL content views
./publish_content_views.py -a -o OtherOrg -d # Dry-run on all OtherOrg views
```

### promote_content_views

Promotes content from the previous lifecycle environment stage.
If a lifecycle is defined as Library -> Test -> Quality -> Production, defining
the target environment (-e) as 'Quality' will promote matching content views
from Test -> Quality.

The following can be promoted:

* Specific content views defined in the main config file
* All content views (-a)

The dry run (-d) option can be used to see what would be promoted for a
given command input. By default progress bars will be displayed for each content
view being promoted. It may be desirable to not display these (e.g. automating), so
they can be disabled using the (-q) option.

Each time a content view is published or promoted, a datestamp is recorded so that
the last publish/promote date can be viewed with the (-l) option. Note that this
datestamp is only updated by this script - it does NOT record publish/promote via
the WebUI or Hammer CLI.

By default the repository metadata is not rebuilt. In some scenarios it may be required
to force a rebuild of the metadata, in which case the (-m) option can be used to trigger this.

The defaults are configured in the main config.yml file in a YAML block like this:

```yaml
promotion:
  batch: 10
  lifecycle1:
    name: Quality
    content_views:
      - RHEL Server
      - RHEL Workstation

  lifecycle2:
    name: Desktop QA
    content_views:
      - RHEL Workstation
```

If multiple lifecycle streams are used in your Satellite installation, the
use of the config.yml definition is strongly recommended to avoid views being
promoted into the wrong lifecycle stream. This is more likely to be an
issue promoting views from the Library, as this is shared by all environments.

The batch: parameter can be used to limit the number of content views that will be promoted at
once, to aid in performance tuning.

#### promote_content_views Help Output

```bash
usage: promote_content_view.py [-h] -e ENV [-o ORG] [-a] [-d] [-m] [-q] [-l]

Promotes content views for specified organization to the target environment.

optional arguments:
  -h, --help         show this help message and exit
  -e ENV, --env ENV  Target Environment (e.g. Development, Quality,
                     Production)
  -o ORG, --org ORG  Organization (Uses default if not specified)
  -a, --all          Promote ALL content views
  -d, --dryrun       Dry Run - Only show what will be promoted
  -l, --last         Display last promotions
  -q, --quiet        Suppress progress output updates
  -m, --forcemeta    Force metadata regeneration
```

#### promote_content_views Examples

```bash
./promote_content_view.py -e Quality            # Promote default views to Quality
./promote_content_view.py -e Production -a      # Promote all views to Production
./promote_content_view.py -e Quality -d         # See what would be done for Quality
```

### auto_content

Sample script that allows for the unattended automation of content management.
This script will find any import datasets present and import them (in order).
Successful import of the content then triggers a publish.  On nominated days/weeks
content is promoted to various lifecycle stages, and content view cleanup is also
performed.  Like the other scripts it calls, it supports a dry run (-d) option to
show what would be performed without actually doing it.

This script can be copied and extended to support custom automation requirements.
