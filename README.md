# This salt runner module is calling SUSE Manager xmlrpc api to patch a system.

salt is the configuration management witin SUSE Manager/Uyuni.
Salt runners are convenience applications executed with the salt-run command.
The runner module here 'suma' has multiple functions. One function named "patch" is written to call \
xmlrpc api of SUSE Manager to schedule a patch job for the given minion system.

## Benefits:
* Trigger a patch job from minion system side. This is often needed to allow admins to execute several tasks within maintenance window consequently. Patch a system is just one step of the tasks.
* Patch job is recorded as history within suse manager.

## Installation on SUSE Manager Server:

```
cd ~/
git clone https://github.com/bjin01/salt-runner-patch.git
cd salt-runner-patch 
cp suma.py to /srv/salt/modules/runners/
```
Now you need to create a new salt-master config file to use this module:
e.g.
```
module_dirs:
  - /srv/saltmodules

runner_dirs:
  - /srv/salt/modules/runners
```

Store the SUSE Manager api login credentials in a file in /etc/salt/master.d/spacewalk.conf
```
# cat /etc/salt/master.d/spacewalk.conf 
spacewalk:
  bjsuma.bo2go.home:
    username: 'bjin'
    password: 'suse1234'
```

Now restart salt-master

## Usage:

__salt execution module:__
This salt-run command utilizes the module suma.patch with a delay of 12 minutes from now:
```salt-run suma.patch suma.bo2go.home testhost01.bo2go.home delay=12```

Or you could also provide a scheduled time for the patch job in the format as below:
```salt-run suma.patch your-suma-host.cool.domain your-minion.great.biz schedule='20:30 24-12-2022'```

__Using salt state__ which utilizes the runner module can be done as below:
```
applypatches:
    runner.suma.patch:
    - server: your-suma-host.cool.domain
    - target_system: {{ data['id'] }}
    - delay: 60
```

Or
```
applypatches:
    runner.suma.patch:
    - server: your-suma-host.cool.domain
    - target_system: {{ data['id'] }}
    - schedule: 21:00 24-12-2022
```

__use the runner module in reactor:__

Define an event to catch in salt-master conf file:
e.g.
```
# cat /etc/salt/master.d/reactor_patchall.conf 
reactor:
  - patching/info/update:
    - /srv/salt/reactor/apply_patches.sls

```

So the event message that salt-master is configured to listen to is "patching/info/update"
Create the reactor state file:
```
# cat /srv/salt/reactor/apply_patches.sls
applypatches:
  runner.suma.patch:
    - server: your-suma-host.cool.domain 
    - target_system: {{ data['id'] }}
    - delay: 5
```

__Test the reactor:__
Run below command on a minion system to send the event:
```
salt-call event.send 'patching/info/update'
```
The event message will be sent to salt-master host which is the SUSE Manager host.
SUSE Manager salt-master will react on this event and run the reactor state defined in /srv/salt/reactor/apply_patches.sls
After few seconds review the scheduled Jobs in SUSE Manager of the minion system in Web UI.

That'S it.
