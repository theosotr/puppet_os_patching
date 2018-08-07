[![Build Status](https://travis-ci.org/albatrossflavour/puppet_os_patching.svg?branch=master)](https://travis-ci.org/albatrossflavour/puppet_os_patching)
# os_patching

This module contains a set of tasks and custom facts to allow the automation of and reporting on operating system patching, currently restricted to Redhat and Debian derivatives.

## Table of contents

<!--TOC-->

## Description

Puppet tasks and bolt have opened up methods to integrate operating system level patching into the puppet workflow.  Providing automation of patch execution through tasks and the robust reporting of state through custom facts and PuppetDB.

If you're looking for a simple way to report on your OS patch levels, this module will show all updates which are outstanding, including which are related to security updates.  Do you want to enable self-service patching?  This module will use Puppet's RBAC and orchestration and task execution facilities to give you that power.

It also uses security metadata (where available) to determine if there are security updates.  On Redhat, this is provided from Redhat as additional metadata in YUM.  On Debian, checks are done for which repo the updates are coming from.  There is a parameter to the task to only apply security updates.

Blackout windows enable the support for time based change freezes where no patching can happen.  There can be multiple windows defined and each which will automatically expire after reaching the defined end date.

## Setup

### What os_patching affects

The module provides an additional fact (`os_patching`) and has a task to allow the patching of a server.  When the `os_patching` manifest is added to a node it installs a script and cron job to generate cache data used by the `os_patching` fact.

### Beginning with os_patching

Install the module using the Puppetfile, include it on your nodes and then use the provided tasks to carry out patching.

## Usage

### Manifest
Include the module:
```puppet
include os_patching
```

More advanced usage:
```puppet
class { 'os_patching':
  patch_window     => 'Week3',
  reboot_override  => true,
  blackout_windows => { 'End of year change freeze':
    {
      'start': '2018-12-15T00:00:00+1000',
      'end': '2019-01-15T23:59:59+1000',
    }
  },
}
```

In that example, the node is assigned to a "patch window", will be forced to reboot regardless of the setting specified in the task and has a blackout window defined for the period of 2018-12-15 - 2019-01-15, during which time no patching through the task can be carried out.

### Task
Run a basic patching task from the command line:
```
os_patching::patch_server - Carry out OS patching on the server, optionally including a reboot and/or only applying security related updates

USAGE:
$ puppet task run os_patching::patch_server [dpkg_params=<value>] [reboot=<value>] [security_only=<value>] [timeout=<value>] [yum_params=<value>] <[--nodes, -n <node-names>] | [--query, -q <'query'>]>

PARAMETERS:
- dpkg_params : Optional[String]
    Any additional parameters to include in the dpkg command
- reboot : Optional[Boolean]
    Should the server reboot after patching has been applied? (Defaults to false)
- security_only : Optional[Boolean]
    Limit patches to those tagged as security related? (Defaults to false)
- timeout : Optional[Integer]
    How many seconds should we wait until timing out the patch run? (Defaults to 3600 seconds)
- yum_params : Optional[String]
    Any additional parameters to include in the yum upgrade command (such as including/excluding repos)
```

Example:
```bash
$ puppet task run os_patching::patch_server reboot=true security_only=false --query="inventory[certname] { facts.os_patching.patch_window = 'Week3' and facts.os_patching.blocked = false and facts.os_patching.package_update_count > 0}"
```

This will run a patching task against all nodes which have facts matching:

* `os_patching.patch_window` of 'Week3'
* `os_patching.blocked` equals `false`
* `os_patching.package_update_count` greater than 0

The task will apply all patches (`security_only=false`) and will reboot the node after patching (`reboot=true`).

## Reference

### Facts

Most of the reporting is driven off the custom fact `os_patching_data`, for example:

```yaml
# facter -p os_patching
{
  package_update_count => 0,
  package_updates => [],
  security_package_updates => [],
  security_package_update_count => 0,
  blocked => false,
  blocked_reasons => [],
  blackouts => {},
  patch_window = 'Week3',
  pinned_packages => [],
  last_run => {
    date => "2018-08-07T21:55:20+10:00",
    message => "Patching complete",
    return_code => "Success",
    post_reboot => "false",
    security_only => "false",
    job_id => "60"
  }
}
```

This shows there are no updates which can be applied to this server.  When there are updates to add, you will see similar to this:

```yaml
# facter -p os_patching
{
  package_update_count => 6,
  package_updates => [
    "kernel.x86_64",
    "kernel-tools.x86_64",
    "kernel-tools-libs.x86_64",
    "postfix.x86_64",
    "procps-ng.x86_64",
    "python-perf.x86_64"
  ]
  security_package_updates => [],
  security_package_update_count => 0,
  blocked => false,
  blocked_reasons => [],
  blackouts => {
    Test change freeze 2 => {
      start => "2018-08-01T09:17:10+1000",
      end => "2018-08-01T11:15:50+1000"
    }
  },
  pinned_packages => [],
  patch_window = 'Week3',
  last_run => {
    date => "2018-08-07T21:55:20+10:00",
    message => "Patching complete",
    return_code => "Success",
    post_reboot => "false",
    security_only => "false",
    job_id => "60"
  }
}
```

Where it shows 6 packages with available updates, along with an array of the package names.  None of the packges are tagged as security related (requires Debian or a subscription to RHEL).  There are no blockers to patching and the blackout window defined is not in effect.

The pinned packages entry lists any packages which have been specifically excluded from being patched, from [version lock](https://access.redhat.com/solutions/98873) on Red Hat or by [pinning](https://wiki.debian.org/AptPreferences) in Debian.

Last run shows a summary of the information from the last `os_patching::patch_server` task.

The fact `os_patching.patch_window` can be used to assign nodes to an arbitrary group.  The fact can be used as part of the query fed into the task to determine which nodes to patch:

```bash
$ puppet task run os_patching::patch_server --query="inventory[certname] {facts.os_patching.patch_window = 'Week3'}"
```

### Task output

If there is nothing to be done, the task will report:

```puppet
{
  "date" : "Mon May 28 12:23:33 AEST 2018",
  "reboot" : "true",
  "message" : "yum dry run shows no patching work to do",
  "return-code" : "success",
  "securityonly" : "false"
}
```

If patching was executed, the task will report:

```puppet
{
  "date": "Mon May 28 12:29:23 AEST 2018",
  "reboot": "true",
  "message": "Patching complete",
  "return-code": "Success",
  "packagesupdated": [
    "kernel-tools-3.10.0-862.2.3.el7.x86_64",
    "kernel-tools-libs-3.10.0-862.2.3.el7.x86_64",
    "postfix-2:2.10.1-6.el7.x86_64",
    "procps-ng-3.3.10-17.el7.x86_64",
    "python-perf-3.10.0-862.2.3.el7.x86_64"
  ],
  "securityonly": "false"
}
```

A summary of the patch run is also written to `/etc/os_patching/run_history`, the last line of which is used by the `os_patching.last_run` fact.

## Limitations

This module is for PE2018+ with agents capable of running tasks.  It is currently limited to the Red Hat and Debian operating system.

## Development

Fork, develop, submit a pull request

## Contributors

- Tony Green <tgreen@albatrossflavour.com>
