This is a rewrite of a userspace USB driver for TEMPer devices presenting
a USB ID like this: `0c45:7401 Microdia`
My device came from [M-Ware ID7747](http://www.m-ware.de/m-ware-usb-thermometer-40--120-c-emailbenachrichtigung-id7747/a-7747/)
and also reports itself as 'RDing TEMPerV1.2'.

Also provides a passpersist-module for NetSNMP (as found in the `snmpd`
packages of Debian and Ubuntu) to present the temperature of 1-3 USB devices
via SNMP.

# Requirements

Basically, `libusb` bindings for python (PyUSB) and `snmp-passpersist` from PyPI.

Under Debian/Ubuntu, treat yourself to some package goodness:

    sudo apt-get install python-usb python-setuptools
    sudo easy_install snmp-passpersist

# Usage

To print temperatures of all sensors found in the system, just run

    python src/temper.py

If your udev installation does not provide access as a normal user to the
USB device, you need to run it as root:

    sudo python src/temper.py

# Serving via SNMP

Using [NetSNMP](http://www.net-snmp.org/), you can use `src/snmp_temper.py`
as a `pass_persist` module.
You can choose one of two OIDs to be emulated: [APC's typical](http://www.oidview.com/mibs/318/PowerNet-MIB.html)
internal/battery temperature (.1.3.6.1.4.1.318.1.1.1.2.2.2.0) or
[Cisco's typical temperature
OIDs](http://tools.cisco.com/Support/SNMP/do/BrowseOID.do?local=en&translate=Translate&objectInput=1.3.6.1.4.1.9.9.13.1.3.1.3)
(.1.3.6.1.4.1.9.9.13.1.3.1.3.1 - 3.3).

Note that you _should not activate both_ modes at the same time.
The reason for this limitation is that the script will keep running for each
`pass_persist` entry and they will interfere with each other when updating the
temperature.
This typically leads to syslog entries like this:

    temper-python: Exception while updating data: could not release intf 1: Invalid argument

## What to add to snmpd.conf

To emulate an APC Battery/Internal temperature value, add something like this to snmpd.conf.
The highest of all measured temperatures in degrees celcius as an integer is reported.

    pass_persist    .1.3.6.1.4.1.318.1.1.1.2.2.2 /path/to/this/script/snmp_temper.py

Alternatively, emulate a Cisco device's temperature information with the following.
The first three detected devices will be reported as ..13.1.3.1.3.1, ..3.2 and ..3.3 .
The value is the temperature in degree celcius as an integer.

    pass_persist    .1.3.6.1.4.1.9.9.13.1.3 /path/to/this/script/snmp_temper.py

Add `--testmode` to the line (as an option to `snmp_temper.py` to enable a mode where
APC reports 99°C and Cisco OIDs report 97, 98 and 99°C respectively. No actual devices
need to be installed but `libusb` and its Python bindings are still required.

## Troubleshooting NetSNMP-interaction

The error reporting of NetSNMP is underwhelming to say the least.
Expect every error to fail silently without a chance to find the source.

`snmp_temper.py` reports some simple information to syslog with an ident string
of `temper-python` and a facility of `LOG_DAEMON`. So this should give you the available debug information:

    sudo tail -f /var/log/syslog | grep temper-python

Try stopping the snmpd daemon and starting it with logging to the console:

    sudo service snmpd stop
    sudo snmpd -f

It will _not_ start the passpersist-process for `snmp_temper.py` immediately
but on the first request for the activated OIDs. This also means that the
first `snmpget` you try may fail like this:

    iso.3.6.1.4.1.9.9.13.1.3.1.3.2 = No Such Instance currently exists at this OID

To test the reporting, try this (twice if it first reports No Such Instance):

    snmpget -c public -v 2c localhost .1.3.6.1.4.1.9.9.13.1.3.1.3.1 # Cisco #1
    snmpget -c public -v 2c localhost .1.3.6.1.4.1.9.9.13.1.3.1.3.2 # Cisco #2
    snmpget -c public -v 2c localhost .1.3.6.1.4.1.9.9.13.1.3.1.3.3 # Cisco #3
    snmpget -c public -v 2c localhost .1.3.6.1.4.1.318.1.1.1.2.2.2.0 # APC

When NetSNMP starts the instance (upon first `snmpget`), you should see something like this in syslog:
    
    Jan  6 16:01:51 raspi-temper1 temper-python: Found 2 thermometer devices.
    Jan  6 16:01:51 raspi-temper1 temper-python: Initial temperature of device #0: 22.2 degree celsius
    Jan  6 16:01:51 raspi-temper1 temper-python: Initial temperature of device #1: 10.9 degree celsius

If you don't even see this, maybe the script has a problem and quits with an exception.
Try running it manually and mimik a passpersist-request (`->` means you should enter the rest of the line):

    -> sudo src/snmp_temper.py 
    -> PING
    <- PONG
    -> get
    -> .1.3.6.1.4.1.318.1.1.1.2.2.2.0
    <- .1.3.6.1.4.1.318.1.1.1.2.2.2.0
    <- INTEGER
    <- 22.25

If you have a problem with the USB side and want to test SNMP, run the script with `--testmode`.

# Origins

The USB interaction pattern is extracted from [here](http://www.isp-sl.com/pcsensor-1.0.0.tgz)
as seen on [Google+](https://plus.google.com/105569853186899442987/posts/N9T7xAjEtyF).
