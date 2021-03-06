QUADS (quick and dirty scheduler)
====================================

Automate scheduling and end-to-end provisioning of R&D scale systems.

**This is moved to https://github.com/redhat-performance/quads**

![quads](/lab-scheduler/image/quads.jpg?raw=true)

**What does it do?**
   - Create and manage a date/time based YAML schedule for machine allocations
   - Drive system provisioning and network switch changes based on workload assignment via external commands
   - Generates instack.json for each OpenStack environment.
   - Automatically generate documentation to illustrate current status, published to a [Wordpress instance](http://python-wordpress-xmlrpc.readthedocs.io/en/latest/examples/posts.html#pages)
     * Current system details
     * Current workloads and assignments
     * Current ownership and resource utilization links (grafana/collectd) 
     * Total duration and time remaining in system assignments
   - Query scheduling data to determine future availability

**Notes**
   - Very simple design (flat files, no external DB)
   - Allows for calling external provisioning commands via ```--path-to-command```

**Workflow**

![quadsworkflow](/lab-scheduler/image/quads-workflow.png?raw=true)

**Scheduler Usage Documentation**
   - Initialize the schedule structure

```
mkdir /etc/lab
./schedule.py --init
```

   - Define the various cloud environments

```
./schedule.py --define-cloud cloud01 --description "Primary Cloud Environment"
./schedule.py --define-cloud cloud02 --description "02 Cloud Environment"
./schedule.py --define-cloud cloud03 --description "03 Cloud Environment"
```

   - Define the hosts in the environment

```
for h in $(hammer host list --per-page 1000 | grep -v mgmt | grep r630 | grep -v c08-h30 | awk '{ print $3 }') ; do ./schedule.py --define-host $h --default-cloud cloud01; done
```

   - To list the hosts:

```
./schedule.py --ls-hosts
```
You will now see the list of full hosts.

```
c08-h21-r630.example.com
c08-h22-r630.example.com
c08-h23-r630.example.com
c08-h24-r630.example.com
c08-h25-r630.example.com
c08-h26-r630.example.com
c08-h27-r630.example.com
c08-h28-r630.example.com
c08-h29-r630.example.com
c09-h01-r630.example.com
c09-h02-r630.example.com
c09-h03-r630.example.com
```

   - To see the current system allocations:

```
./schedule.py --summary
cloud01 : 45 (Primary Cloud Environment)
cloud02 : 0 (02 Cloud Environment)
cloud03 : 0 (03 Cloud Environment)
```

   - Sync states of each host.
     - This needs to be done whenever a new host is created.
     - We also need to track the last configured environment of each host (this is how we track whether or not we need to reconfigure a host if the schedule changes).
     - *Note*: state files are stored in ```/etc/lab/state/HOSTNAME``` for each host and contains the current cloud membership

```
./schedule.py --sync
```

   - Define a custom schedule for a host
     - Example: assign host ```c08-h21``` to the workload/cloud ```cloud02```

```
./schedule.py --add-schedule --host c08-h21-r630.example.com --schedule-start "2016-07-11 08:00" --schedule-end "2016-07-12 08:00" --schedule-cloud cloud02
```

   - List the schedule for a specific host:

```
./schedule.py --ls-schedule --host c08-h21-r630.example.com
```

You'll see the schedule output below

```
Default cloud: cloud01
Current cloud: cloud02
Defined schedules:
  0:
    start: 2016-07-11 08:00
    end: 2016-07-12 08:00
    cloud: cloud02
```

   - Move any hosts that need to be re-allocated based on the current schedule

```
./schedule.py --move-hosts
```

You should see the following verbosity from a move operation

```
INFO: Moving c08-h21-r630.example.com from cloud01 to cloud02 c08-h21-r630.example.com cloud01 cloud02
```

In the above example the default move command was called ```/bin/echo``` for illustration purposes.  In order for this to do something more meaningful you should invoke the script with the ```--move-command``` option, which should be the path to a valid command.  The script takes three arguments (hostname, current cloud, new cloud).


   - Move a host using --move-command
     - You can append a script, command or other action as a post-hook (perhaps to fire off system provisioning).

```
./schedule.py --move-hosts --path-to-command /usr/bin/movecommand.sh
```
