# junos-ephemeral-save

Example Junos scripts to save an ephemeral database to disk and restore it manually or at startup.
The config is saved in Junos format for size efficiency compared to saving and restoring in XML format.

## Todo

- No error handling done for now. 
- Support for redundant Routing Engines with failover detection needs 
an additional event trigger and the save op script must copy the saved file also to the backup 
routing-engine.

## Requirements

Junos physical or virtual device with ephemeral database support (e.g. 16.1 or newer).

## Installation

Upload the op scripts to /var/db/scripts/op/ and the event script into /var/db/scripts/event/ to
the Junos device (hostname oc1 in the example here):

```
$ scp save-ephemeral.slax load-ephemeral.slax oc1:/var/db/scripts/op/
$ scp event-load-ephemeral.slax oc1:/var/db/scripts/event/
```

Configure Junos to accept the op and event script and trigger the script at startup based on the event SNMPD_TRAP_COLD_START:

```
mwiget@oc1> show configuration system scripts
op {
  file load-ephemeral.slax;
  file save-ephemeral.slax;
}

mwiget@oc1> show configuration event-options
policy ephemeral-restore-state {
  events SNMPD_TRAP_COLD_START;
  then {
    event-script event-load-ephemeral.slax;
  }
}
event-script {
  file event-load-ephemeral.slax;
}
```

## Test

Use netconf XML API to add a configuration into the ephemeral database (not shown here):

```
mwiget@oc1> show ephemeral-configuration 0
## Last changed: 2016-12-02 13:21:51 UTC
interfaces {
  ge-0/0/1 {
    unit 0 {
      family inet {
        address 192.168.168.1/24;
      }
    }
  }
}
```

Use the save op script to save its content to a file on the device:

```
mwiget@oc1> op save-ephemeral
ephemeral db saved to /var/db/scripts/lib/ephemeral0.txt
```

Optionally verify the content of the created file with 'file show':

```
mwiget@oc1> file show /var/db/scripts/lib/ephemeral0.txt

## Last changed: 2016-12-02 13:21:51 UTC
interfaces {
  ge-0/0/1 {
    unit 0 {
      family inet {
        address 192.168.168.1/24;
      }
    }
  }
}
```

Reboot now the device and find the ephemeral database be restored autoamtically. The messages log file
will show the following entry if successful:

```
Dec  2 13:22:01  oc1 snmpd[6067]: SNMPD_TRAP_COLD_START: trap_generate_cold: SNMP trap: cold start
Dec  2 13:22:01  oc1 mgd[6084]: UI_EPHEMERAL_COMMIT: User 'root' has requested commit on '0' ephemeral database
Dec  2 13:22:01  oc1 mgd[6084]: UI_EPHEMERAL_COMMIT_COMPLETED: commit complete on '0' ephemeral database
Dec  2 13:22:01  oc1 cscript.crypto: event-load-ephemeral.slax: ephemeral db 0 restored from file /var/db/scripts/lib/ephemeral0.txt

```

Verify the content of the ephemeral database:

```
mwiget@oc1> show ephemeral-configuration 0
## Last changed: 2016-12-02 13:21:51 UTC
interfaces {
  ge-0/0/1 {
    unit 0 {
      family inet {
        address 192.168.168.1/24;
      }
    }
  }
}
```



