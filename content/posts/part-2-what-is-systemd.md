---
title: "Part 2 What Is Systemd"
date: 2023-11-30T20:46:08+01:00
draft: false
---

# Part 2 - What is systemd?
I already know what systemd is and what is does. But I want to learn more about the configuration
options it have or get a better understanding of the file, which I created for the service.

There are a lot more keywords that can be used in that file to make systemd do other things.

## Options
There are a lot options. In the book I'm reading, there are listed the following options as the common
ones that will cross your way regarding service untis:

### Description
Obviously the description of the service.

### Documentation
If there is any intersting or helpful information you can place links or references here.

### After
Used to define dependencies. So this service should be started after the one described in this option.

### Before
Same as after, just before.

### Wants
The services in **Wants** should be successfully started or run.

### Require
The option **Wants** is the moderated version of this one.
If the required services are stopped, our service should also stop.

### EnvironmentFile
This is used to reference a file with environment variables.

### ExecStart
Contains the command that is used when the service is started.

### ExecReload
With this command the configuration files are reloaded.

### ExecStop
Contains the command that is used to stop.

### KillMode
Shows with which procedure the the process can be killed

### Restart
Defines the options for an automatic restart.

### RestartSec
Holds the time after the service should restart.

### WantedBy
Describes the target or service through the service should be started automatically.

### Conflicts
Stops the conflicting services, if our service is started.

### OnFailure
Holds a list of services that will be started, when our service fails.

### Type
Has multiple options: Can be `simple` or `forking`. The first one is a process
that is always running. The second one is forking a child process. `oneshot` is another
option, which will only run once.

# Target Units
After I discovered the service units, it's time to learn about target units. What are target units?

An easy explanation is: Targets are different states that the system can boot into. So there are different
entrypoints for your service, since I can put the target into the WantedBy option and my service will start after 
this state is reached.

To get a list of the available targets I use the command: `systemctl list-units '*.target` and this list is returned:

```
 UNIT                    LOAD   ACTIVE SUB    DESCRIPTION                        
  basic.target            loaded active active Basic System
  cloud-config.target     loaded active active Cloud-config availability
  cloud-init.target       loaded active active Cloud-init target
  cryptsetup.target       loaded active active Local Encrypted Volumes
  getty-pre.target        loaded active active Preparation for Logins
  getty.target            loaded active active Login Prompts
  graphical.target        loaded active active Graphical Interface
  local-fs-pre.target     loaded active active Preparation for Local File Systems
  local-fs.target         loaded active active Local File Systems
  multi-user.target       loaded active active Multi-User System
  network-online.target   loaded active active Network is Online
  network-pre.target      loaded active active Preparation for Network
  network.target          loaded active active Network
  nss-lookup.target       loaded active active Host and Network Name Lookups
  nss-user-lookup.target  loaded active active User and Group Name Lookups
  paths.target            loaded active active Path Units
  printer.target          loaded active active Printer Support
  remote-fs-pre.target    loaded active active Preparation for Remote File Systems
  remote-fs.target        loaded active active Remote File Systems
  slices.target           loaded active active Slice Units
  snapd.mounts-pre.target loaded active active Mounting snaps
  snapd.mounts.target     loaded active active Mounted snaps
  sockets.target          loaded active active Socket Units
  sound.target            loaded active active Sound Card
  swap.target             loaded active active Swaps
  sysinit.target          loaded active active System Initialization
  time-set.target         loaded active active System Time Set
  timers.target           loaded active active Timer Units
  veritysetup.target      loaded active active Local Verity Protected Volumes

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type.
29 loaded units listed. Pass --all to see loaded but inactive units, too.
To show all installed unit files use 'systemctl list-unit-files'.
```

I also learned a new command to list all dependencies: `systemctl list-dependencies mydashboard`. It lists all dependencies 
for the service.

# Next part
I already could use it on my server to create a service for a python script
which I would restart manually before I knew that I could use a service. 

Feels really good! Next post will be a bit about journald, which also belongs to systemd and is used
for logging.