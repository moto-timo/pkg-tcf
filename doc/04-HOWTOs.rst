======
HOWTOs
======

General Linux system
====================

.. _setup_proxies:

Setup proxies
-------------

If your network requires proxy support:

- From the GUI: log in as a normal user to the graphical interface

  1. On the top right click the configuration arrow, select * →
     configuration icon → select network → select proxies*

  2. Set:

     - Method *Manual*

     .. include:: 04-HOWTOs-LL-proxy-1.rst

- From the terminal, (when remotedly logged in, optional) add to
  ``~/.bashrc`` or to ``/etc/bashrc`` (for system wide, headless
  machines):

    .. literalinclude:: 04-HOWTOs-LL-proxy-2.sh
       :language: shell

  (note the ``${VAR:-DEFAULT}`` syntax is so to avoid these settings
  to interfere with those that might be set when you login via the GUI).

Configure *sudo* to pass proxy configuration::

  # cat <<EOF > /etc/sudoers.d/proxy
  # Keep proxy configuration through sudo, so we don't need to specify -E
  # to carry it
  Defaults env_keep += "ALL_PROXY FTP_PROXY HTTP_PROXY HTTPS_PROXY NO_PROXY"
  Defaults env_keep += "all_proxy ftp_proxy http_proxy https_proxy no_proxy"
  EOF

This ensures the proxy configuration is kept by *sudo* (versus having
torun *sudo -E*) so tools that use the network don't get stuck with
network access.

Why?

.. include:: 04-HOWTOs-LL-proxy-3.rst

- *127.0.0.1/8* and *192.168.0.0/16* will be all networks we'll use
  internally from our server, so they don't need to be proxyed.

.. _tcf_update:

How do I update my TCF installation?
------------------------------------

You can check if there is a new version available with::

  # dnf check-update --refresh | grep TCF

if so, you can update to it with::

  # dnf update --best ttbd ttbd-zephyr tcf tcf-zephyr
  # systemctl restart ttbd@production

Note that if there is a major version change (from *v0* to *v1*, or
*v1* to *v2*, etc), more steps have to be done, which will be like
(example is from *v0* to *v1*)

.. include:: 04-HOWTOs-LL-update.rst

This is so because we release the TCF stable branches as separate RPM
repositories, for simplicity.

Note you might need to clean the metadata caches with::

  # dnf clean all

to force an update of the metadata to be pulled from the servers.

.. _internal_network:

Configuring a USB or other network adapter for an internal network
------------------------------------------------------------------

Test targets or infrastructure (like power control switches) that
require IP connectivity can be connected to your server via a
dedicated network interface and switch.

.. warning:: do not connect infrastructure and test devices to the
             same network! You have to :ref:`keep them separated
             <separated_networks>`.

You will need:

- a network interface
- its MAC address
- an IP address range (recommended *192.168.X.0/24*)
- a name for the interface (recommended *ngX*, with X matching the
  network range above)

Once a new interface is identified (builtin, PCI card or USB),
identify its name and MAC address::

  $ ifconfig -a		# to list all interfaces

Once you have identified the interface which is to be used, find out
it's MAC address::

  $ ifconfig enp0s29u1u3	# To list information about an  specific interface
  enp0s29u1u3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.1  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::210:60ff:fe31:a4ba  prefixlen 64  scopeid 0x20<link>
        ether 00:10:60:31:a4:ba  txqueuelen 1000  (Ethernet)
        RX packets 68626  bytes 3156948 (3.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 76284  bytes 13537889 (12.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

in this case, the fourth line of output, *ether 00:10:60:31:a4:ba*.

Now if we are going to assing it the network range *192.168.3.0/24*,
we assign it name *ng3*.

With the MAC address (*00:10:60:31:a4:ba*), a name (*ng3*), an IP
network (*192.168.3.0*) and it's range (*24*), we can create a
configuration file::

  # cat > /etc/sysconfig/network-scripts/ifcfg-ng3 <<EOF
  TYPE=Ethernet
  BOOTPROTO=none
  ONBOOT=yes
  PREFIX=24
  IPADDR=192.168.3.1
  HWADDR=00:10:60:31:a4:ba
  EOF

Note the server is give address **.1** in the *192.168.3.0/24* network.

And start it::

  # ifup ng3

Now you can connect this adaptor to a network switch and connect
clients to said switch.

.. warning:: Keep this switch isolated from upstream routers; connect
             only test-targets OR infrastructure elements to it.

.. _howto_nm_disable_control:

Disabling NetworkManager from controlling the interface
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

NetworkManager needs to be told to ignore the network device; for
that and our current example, run::

 $ sudo tee /etc/NetworkManager/conf.d/disabled.conf <<EOF
 [keyfile]
 unmanaged-devices=mac:00:10:60:31:a4:ba
 EOF

Or edit said file and add *mac:MACADDR* statements
separated by semicolons to the *unmanaged-devices* line.

.. _howto_nm_config_static:

Configuring a static interface Via NetworkManager
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For example, to configure an internal network interface *IFNAME* for
internal network for infrastructure control *192.168.2.0/24* with IP
*.205* (for testcase execution we need Network Manager :ref:`off the
interface <howto_nm_disable_control>`), run::

  # nmcli con add type ethernet con-name tcf-internal-2 ifname IFNAME ip4 192.168.2.205/24
  # nmcli con up tcf-internal-2

.. _generate_ssl_certificate:

Generating an SSL certificate
-----------------------------

To use secure layers HTTPS connections between daemon and client, you
must have a valid certificate and key, or use an autosigned
certificate. This can be fed to the broker server with the options
``--ssl-crt`` and ``--ssl-key``.

If  you want to create your own certificate you must have installed
OpenSSL:

1. Generate a private key::

   $ openssl genrsa -des3 -out server.key 1024

2. Generate a CSR::

   $ openssl req -new -key server.key -out server.csr

3. Remove Passphrase from key::

   $ cp server.key server.key.org
   $ openssl rsa -in server.key.org -out server.key

4. Generate self signed certificate::

   $ openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt

 Instructions taken from:
 http://kracekumar.com/post/54437887454/ssl-for-flask-local-development

.. _find_usb_info:

Finding USB device information
------------------------------

To find information about USB devices (serial numbers, paths) use any
of these methods.

Note it is also possible that a device declares a serial number but it
is all the same for every device (for example, in the FlySwatter2 and
other FTDI based hardware). In FTDI based case, it is possible to
re-flash a new serial number with :ref:`these instructions
<fs2_serial_update>`

.. note:: use these methods in a Linux machine that is not running
          *TTBD* actively, as it will keep producing messages in the
          kernel output and it will be difficult to tell which device
          is the right one.

Finding USB device information with dmesg
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

1. disconnect the device
2. run `dmesg -w` in a console to see the kernel log, hit ``enter`` a
   few times to create space
3. plug the device, see a message pop up with device data from the
   kernel
4. Note the device data, along::

     usb 1-1.4.4.4.1: new full-speed USB device number 62 using ehci-pci
     usb 1-1.4.4.4.1: New USB device found, idVendor=2a03, idProduct=003d
     usb 1-1.4.4.4.1: New USB device strings: Mfr=1, Product=2, SerialNumber=220
     usb 1-1.4.4.4.1: Product: Arduino Due Prog. Port
     usb 1-1.4.4.4.1: Manufacturer: Arduino (www.arduino.org)
     usb 1-1.4.4.4.1: SerialNumber: 85439303033351E06162

   in this example, ignore the *SerialNumber* where it says *New USB
   device Strings* and focus on the following one.

   If there is no serial number, there will be something like::

     New USB device strings: Mfr=XYZ, Product=XYZ, SerialNumber=0

   and no line with just *SerialNumber* on its own will appear.

Finding USB device information with lsusb.py
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``lsusb.py -iu`` provides a tree display of the USB connected devices::

  usb1            1d6b:0002 09  2.00  480MBit/s 0mA 1IFs (ehci_hcd 0000:00:1a.7) hub
   1-1            05e3:0608 09  2.00  480MBit/s 100mA 1IFs (Genesys Logic, Inc. Hub) hub
    1-1.1         2001:f103 09  2.00  480MBit/s 0mA 1IFs (D-Link Corp. DUB-H7 7-port USB 2.0 hub) hub
     1-1.1.1      0424:2514 09  2.00  480MBit/s 2mA 1IFs (Standard Microsystems Corp. USB 2.0 Hub) hub
      1-1.1.1.1   2a03:003d 02  1.10   12MBit/s 100mA 2IFs (Arduino (www.arduino.org) Arduino Due Prog. Port 85439303033351E01192)
     1-1.1.2      0424:2514 09  2.00  480MBit/s 2mA 1IFs (Standard Microsystems Corp. USB 2.0 Hub) hub
      1-1.1.2.4   04d8:f2f7 00  2.00   12MBit/s 100mA 1IFs (Yepkit Lda. YKUSH YK20946)
       1-1.1.2.4:1.0(IF) 03:00:00 2EPs (Human Interface Device:No Subclass:None)
     1-1.1.5      0403:6001 00  2.00   12MBit/s 90mA 1IFs (FTDI FT232R USB UART A5026SO1)

if we know the brand of our device, we can look it up; in the excerpt
tree below, we can see, for example, a YKUSH on *1-1.1.2.4*; next to
the name of the device is the serial number (if available), *YK20946*
(in this example).

Finding USB device information with udevadm
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once we know a device node of any time has been established (eg:
:file:`/dev/ttyUSB9`), use ``udevadm info`` to find information about
it::

  # udevadm info /dev/ttyUSB0	# or whichever device node it is
  P: /devices/pci0000:00/0000:00:14.0/usb1/1-2/1-2:1.0/ttyUSB0/tty/ttyUSB0
  N: ttyUSB0
  S: serial/by-id/pci-Prolific_Technology_Inc._USB-Serial_Controller-if00-port0
  S: serial/by-path/pci-0000:00:14.0-usb-0:2:1.0-port0
  ...
  E: ID_SERIAL_SHORT=OR0497598
  E: ID_PATH=pci-0000:00:14.0-usb-0:2:1.0
  E: ID_PATH_TAG=pci-0000_00_14_0-usb-0_2_1_0
  E: ID_PCI_CLASS_FROM_DATABASE=Serial bus controller
  E: ID_PCI_INTERFACE_FROM_DATABASE=XHCI
  E: ID_PCI_SUBCLASS_FROM_DATABASE=USB controller
  ...

those values can be used to match in udev

.. _usb_tty:

udev configuration of serial port
---------------------------------

Methods for configuring a serial port by name. When a USB serial port
is connected to the system, it is assigned a non-predictable name (eg:
*/dev/ttyUSB14* or */dev/ttyACM3*) which will change each time is
plugged (or not).

In order to have an stable name that consistently represents the
device we are connecting, follow any of the following recipes, based
on the capabilities of the USB device you are plugging to the system.

.. _usb_tty_serial:

udev configuration of serial port based on serial number
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Given a target name (TARGETNAME) we are going to name a serial port
assigned to it `/dev/tty-TARGETNAME` based on the *serial number* of
the USB device (that can be found using the :ref:`tricks above
<find_usb_info>`) in :file:`/etc/udev/rules.d/90-ttbd.rules`::

  SUBSYSTEM == "tty", ENV{ID_SERIAL_SHORT} == "SERIALNUMBER", \
    SYMLINK += "tty-TARGETNAME"

Remember to update udev::

    # udevadm control --reload-rules

.. _usb_tty_path:

udev configuration of serial port without serial number based on path
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is used for USB serial dongles that have no unique USB serial number.

.. warning:: This approach is very risky--any physical changes in the
             positions of cables or logical changes in enumeration
             order when kernel boots will change the path and might
             render your configuration inoperative or working incorrectly.

Given a target name (TARGETNAME) we are going to name a serial port
assigned to it `/dev/tty-TARGETNAME` based on the *path* the USB
device is connected, as most serial cables have no serial number; in
:file:`/etc/udev/rules.d/90-ttbd.rules`::

  SUBSYSTEM == "tty", ENV{ID_PATH} == "*-usb-0:1.2.2:1.0", \
    SYMLINK += "tty-TARGETNAME"

find the path by plugging the device, using `dmesg -w` to find which
`/dev/ttyUSBX` (or `/dev/ttyACMX`) name is given and then running::

  # udevadm info /dev/ttyUSBX | grep ID_PATH=
  ID_PATH=pciblahblah-usb-0:1.2.2:1.0

reload udev configuration::

  # udevadm control --reload-rules

replug the USB dongle or device and verify the symlink
`/dev/tty-TARGETNAME` is there.

If not found, use syslog or `journalctl -r` and `udevadm
control --log-level debug` to find our what is going on when the
device is plugged.


.. _usb_tty_sibling:

udev configuration of serial port based on sibling's serial number
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This is used for USB-to-TTY serial dongles that have no unique USB
serial number.

Given a target name (TARGETNAME) we are going to name a USB-to-TTY
serial port assigned to it `/dev/tty-TARGETNAME` based on the USB
serial number of another device (the sibling) connected to the same
USB hub.

The sibling USB device has to have a unique USB serial number. By
knowing in which port our the USB-to-TTY serial dongle is, we can
piggy back on the other sibling device's USB serial number.

Thus, if we move the USB hub around, it'll still have the same name,
as long as we don't change the port to which it is connected.

In :file:`/etc/udev/rules.d/90-ttbd.rules`::

  # When the USB-TTL port is connected to the hub with serial number
  # YKXXXXX
  SUBSYSTEM == "tty", \
      PROGRAM = "/usr/bin/usb-sibling-by-serial YKXXXXX", \
      ENV{ID_PATH} == "*2:1.0", \
      SYMLINK += "tty-TARGETNAME"

note the ``*2:1.0``; that selects port 2, where we connect the
serial port adapter. Port 1 would be ``*1:1.0``, etc.

Remember to update udev::

  # udevadm control --reload-rules

.. _usbrly08b_serial_number:

Finding the serial number of a Devantech USBRLY08B relay controller
-------------------------------------------------------------------

Plug your board to the system and list::

  $ lsusb.py | grep -i Devantech
   1-1.1.1       04d8:ffee 02  2.00   12MBit/s 100mA 2IFs (Devantech Ltd. USB-RLY08 00023456)

*00023456* is the serial number; if there are multiple devices and you
are not sure which one is it, disconnect the one you care for and run::

  $ lsusb.py | grep -i Devantech > before

now reconnect it and run::

  $ lsusb.py | grep -i Devantech > after

diff the files *before* and *after*::

  $ diff before after
  3d2
  <  1-1.1.1       04d8:ffee 02  2.00   12MBit/s 100mA 2IFs (Devantech Ltd. USB-RLY08 00023456)

or run `dmesg -w` on the terminal, unplug and plug the device, see the
kernel message about the new USB device, note the serial number.


.. _ykush_serial_number:

Finding the serial number of an YKUSH hub
-----------------------------------------

Methods to find the serial number of an YKUSH hub:

- plug your YKush hub to the system and list::

    # lsusb.py | grep YKUSH | tee before
      2-2.1.1.4    04d8:f2f7 00  2.00   12MBit/s 100mA 1IF  (Yepkit Lda. YKUSH YK21297)
      2-2.1.2.4    04d8:f2f7 00  2.00   12MBit/s 100mA 1IF  (Yepkit Lda. YKUSH YK21292)
      2-2.1.3.4    04d8:f2f7 00  2.00   12MBit/s 100mA 1IF  (Yepkit Lda. YKUSH YK21290)
      2-2.1.4.4    04d8:f2f7 00  2.00   12MBit/s 100mA 1IF  (Yepkit Lda. YKUSH YK21294)
     2-2.3.4       04d8:f2f7 00  2.00   12MBit/s 100mA 1IF  (Yepkit Lda. YKUSH YK21293)

  you might have many; to tell which one is the one in your hand, you
  can just unplug it and list again::

    # lsusb.py | grep YKUSH | tee after
     2-2.1.1.4    04d8:f2f7 00  2.00   12MBit/s 100mA 1IF  (Yepkit Lda. YKUSH YK21297)
     2-2.1.2.4    04d8:f2f7 00  2.00   12MBit/s 100mA 1IF  (Yepkit Lda. YKUSH YK21292)
     2-2.1.4.4    04d8:f2f7 00  2.00   12MBit/s 100mA 1IF  (Yepkit Lda. YKUSH YK21294)
    2-2.3.4       04d8:f2f7 00  2.00   12MBit/s 100mA 1IF  (Yepkit Lda. YKUSH YK21293)

  to (quickly) find out which one was unplugged, diff the files
  *before* and *after*::

    # diff before after
    3d2
    <     2-2.1.3.4    04d8:f2f7 00  2.00   12MBit/s 100mA 1IF  (Yepkit Lda. YKUSH YK21290)

  the line that was removed when we unplugged was the one for *YK21290*,
  which is the serial number of the hub.

- run `dmesg -w` on a terminal and plug the device, see the kernel
  message about the new USB device, note the serial number

.. note:: the hub itself has no serial number, but an internal device
          connected to its downstream port number 4 does have the
          *YK34567* serial number.

*ttbd*: TCF server configuration and tricks
===========================================

.. _ttbd_config_multiple_instances:

Starting more that one instance
-------------------------------

Most setups will only have one instance, the *production* instance;
however, two more are recommended:

- *infrastructure*: handles power to USB hubs, equipment, normal and
  power switching to reset them in case) of issues, individual raw
  access to all the power switching units connected throughout the
  system, network switches, etc

- *staging*: targets whose drivers are being developed before moving
  into production (optional)

To bring up an instance (repeat these steps replacing *production* with
*INSTANCENAME* for other instances):

1. Create the instance's configuration directory::

     # install -d -m 2775 -o ttbd -g ttbd /etc/ttbd-production

   systemd will create the runtime directories needed when starting
   *ttbd* in `/var/run/ttbd-production` and
   `/var/cache/ttbd-production` as they will be wiped by the system
   on restart.

2. Each instance listens on a different port, so we create the initial
   server configuration `/etc/ttbd-production/conf_00_bind.py`::

     host = "0.0.0.0"		# Listen on all interfaces
     port = 5000

   *infrastructure* is usually assigned 4999, *staging* 5001. Enable
   access through the firewall to said ports::

     # firewall-cmd --add-port=4999-5001/tcp --permanent

4. Enable and start the instance::

     # systemctl enable ttbd@production
     # systemctl start ttbd@production

   *systemd* runs the daemon run as user *ttbd*, group *ttbd* and
   the following supplemental groups:

     - *root*: to be able to scan USB devices
     - *dialout*: to be able to access serial devices and USB connected
       serial devices (for consoles, JTAGs, etc)
     - *ttbd*: to access configuration and other files

   See log output with ``journalctl -fu ttbd@NAME``. Diagnose issues
   starting with systemd in :ref:`troubleshooting
   <systemd_tips_diagnosis>`. Further configuration :ref:`tips
   <systemd_tips_configuring>`.

5. At this point you could create or copy existing ``conf_*.py``
   configuration files to ``/etc/ttbd-production`` and restart the
   service with::

     # systemctl restart ttbd@production

   If you have none, that is ok, we'll add them in the next sections.

Note that now, none can access the server yet because there is no way
to authenticate with it :) Let's add some configuration.

.. _ttbd_config_auth_local:

Configure authentication for local users (optional)
---------------------------------------------------

A quick way to allow any user in the local machine to use the server
without authenticating is to request local authentication for
127.0.0.1; run::

  # echo 'local_auth.append("127.0.0.1")' \
    > /etc/ttbd-production/conf_00_auth_local.py
  # systemctl restart ttbd@production

Now login in should work with no need to input anything (in this case,
there will be no output either)::

  $ tcf -iu https://localhost:5000 login

.. note:: feel free to ignore the error message about `ZEPHYR_BASE not
          being defined`; it is a glitch that will be fixed. You can
          work around it by running::

            $ export ZEPHYR_BASE=

You can configure local TCF clients to access the local instance
by default (:ref:`more <tcf_configuring>`)::

  # mkdir -p /etc/tcf
  # echo "tcfl.config.url_add('https://localhost:5000', ssl_ignore = True)" \
    >> /etc/tcf/conf_local.py


.. _ttbd_config_authdb:

Configure simple authentication / for Jenkins jobs (optional)
-------------------------------------------------------------

You can setup static accounts for users or Jenkins autobuilders with
hardcoded passwords by creating a file
``/etc/ttbd-production/conf_00_auth_localdb.py`` with the contents:

.. code-block:: python

   import ttbl.auth_localdb

   ttbl.config.add_authenticator(ttbl.auth_localdb.authenticator_localdb_c(
        "Jenkins and other",
        [
            [ 'usera', 'PASSWORDA', 'user', ],
            [ 'superuserB', 'PASSWORDB', 'user', 'admin', ],
            [ 'jenkins1', 'PASSWORD1', 'user', ],
            [ 'jenkins2', 'PASSWORD2', 'user', ],
            ...
        ]))

Restart to read the configuration::

  # systemctl restart ttbd@production

.. _ttbd_config_auth_ldap:

.. include:: 04-HOWTOs-LL-auth-ldap.rst

Configure authentication against LDAP
-------------------------------------

Copy the configuration example
`/etc/ttbd/production/example_conf_05_auth_ldap.py` and modify it to
suit your LDAP setup::

  # cp /etc/ttbd-production/example_conf_05_auth_ldap.py /etc/ttbd-production/conf_05_auth_ldap.py
  # <edit away>
  # systemctl restart ttbd@production

Authorized users (users in the declared LDAP groups in the
configurtion file) should now be able to login with::

  $ tcf login LDAPLOGIN

This file can be tweaked to fit your authentication needs. For
example, you might want to change the name of the LDAP groups your
users need to be members off.

If this does not work, most likely the configuration files where not
loaded properly; check the daemon output (:ref:`troubleshooting
<systemd_tips_diagnosis>` and :ref:`troubleshooting LDAP
<ttbd_auth_ldap_invalid_creds>`).



.. _tt_power:

Configuring a target for just controlling power to something
------------------------------------------------------------

In your *ttbd*'s `conf_SOMETHING.py` config file, add::

  ttbl.config.target_add(
      ttbl.tt.tt_power(
        "TARGETNAME",
        power_control = PC_OBJECT,
        power = True
      ),
      tags = dict(idle_poweroff = 0)
  )

this creates a target called *TARGETNAME* that you can power on, off
or cycle. The *power* argument indicates what do do with the power at
startup (*True* turn it on, *False* turn it off, *None*--or omitting
this argument, leave it as is). *tags = dict(idle_poweroff = 32)* is
used to have *TARGETNAME* not being powered off when idle.

Now, *PC_OBJECT* is the actual implementation of power control and you
can make it be things like::

  ttbl.pc.dlwps7("http://admin:1234@sp3/5")

This would make *TARGETNAME*'s power be controlled by plug #5 of the
*Digital Logger Web Power Switch 7* named *sp3* (:py:func:`setup
instructions <conf_00_lib.dlwps7_add>`). Because this is a normal,
120V plug, if a light bulb were connected to it::

    ttbl.config.target_add(
      ttbl.tt.tt_power(
        "Entrance_light",
        power_control = ttbl.pc.dlwps7("http://admin:1234@sp3/5"),
        power = True
      ),
      tags = dict(idle_poweroff = 0)
    )

and like this, the target *Entrance_light* can be switched on or off
with *tcf power-on Entrance_light* and *tcf power-off Entrance_light*.

It could also be::

  ttbl.pc_ykush.ykush("YK21080", 3)

which means that power to *TARGETNAME* would be implemented by
powering on or off port #3 of the YKush power-switching hub with
serial number *YK21080* (:py:func:`setup instructions
<conf_00_lib.ykush_targets_add>`).

Other power controller implementations are possible, of course, by
subclassing :py:class:`ttbl.tt_power_control_impl`.

.. _target_tag:

Adding tags to a target
-----------------------

Once a target is configured, new tags can be added to it using
:meth:`ttbl.test_target.tags_update` in any configuration file
(preferrably next to the target definition); for example, if in
``/etc/ttbd-production/conf_10_targets.py`` you had the statement:

.. code-block:: python

   arduino101_add(name = "a101-15",
                  fs2_serial = "a101-15-fs2",
                  ykush_url = "http://admin:1234@HOSTNAME/PORT",
                  ykush_serial = "YK24439")

you can add the tags ``fixture_spi_basic_0`` (as a boolean that
defaults to *True* and  ``tempsetting`` (as an integer) with:

.. code-block:: python

   arduino101_add(name = "a101-15",
                  fs2_serial = "a101-15-fs2",
                  ykush_url = "http://admin:1234@HOSTNAME/PORT",
                  ykush_serial = "YK24439")
   tcfl.config.targets['a101-15'].tags_update(dict(
       fixture_spi_basic_0 = True,
       tempsetting = 32))

.. _target_disable_default:

How do I disable a target by default?
-------------------------------------

When a target has to be disabled by default, add this to the
configuration file::

  ttbl.config.targets['TARGETNAME'].disable("")

the target will be loaded and the configuration will be accesible,
however, *tcf* clients that select targets automatically (*list*,
*run*) will not use it unless *-a* is given.

This is used for targets that are misbehaving for any reason but still
need to be connected to debug. They can be manually enabled/disabled
with::

  $ tcf disable TARGETNAME
  $ tcf enable TARGETNAME

The user has to have *admin* capabilities in the *TTBD* server to run
this operation.

.. _ttbd_config_bind:

Allowing the server to be used remotely
---------------------------------------

By default, the server is configured to only listen on local ports,
thus only accessible from the server itself.

To allow the server to be accessible on all the network interfaces of
the machine on TCP port 5000, create
``/etc/ttbd-production/conf_00_bind.py`` with the content:

.. code-block:: python

   host = "0.0.0.0"
   port = 5000

Now restart the daemon and verify it restarted properly::

  # systemctl restart ttbd@production
  # journalctl -eu ttbd@production

As well, ensure the server's firewall allows the given ports to be
accessible. In Fedora 25::

  # dnf install -y firewall-config
  $ firewall-config

In the current firewall zone, add in *Ports* a range *4999-5001*, type
TCP.

Now select in the menu *Options* > *Runtime to permanent* to ensure
the changes are permanent and next time the server restarts they are
applied.


Increasing the verbosity of the server
--------------------------------------

When debugging issues with the server, you might have to increase its
verbosity; for that, more *-v* have to be given to the command
line. For that, edit ``/etc/systemd/system/ttbd@.service`` to add more
``-v`` to the ``ExecStart`` line.

Reread *systemd*'s configuration and restart the server::

  # systemctl daemon-reload
  # systemctl restart ttbd@production


Remember to toggle it back to the default ``-vv``--it gets chatty.


TCF client tricks
=================


How do I release a target I don't own?
--------------------------------------

Someone owns the target and they have gone home...::

  $ tcf release -f TARGETNAME

But this only works if you have admin permissions.

The exception is if you have locked yourself the target with a
*ticket* (used by *tcf run* and others so that the same user running
different processes in parallel can still exclude itself from
overusing a target). It will say something like::

  requests.exceptions.HTTPError: 400: TARGETNAME: tried to use busy target (owned by 'MYUSERNAME:TICKETSTRING')

As a user, you can always force release any of your own locks with
`-f`.


How can I debug a target?
-------------------------

TCF provides for means to connect remote debuggers to targets that
support them; if the target supports the
:py:class:`ttbl.tt_debug_mixin` (which you can find with `tcf list -vv
TARGETNAME | grep interfaces`).

How it is done and what are the capabilities depends on the target,
but in general, assuming you have a target with an image deployed::

  $ tcf acquire TARGETNAME
  $ tcf debug-start TARGETNAME
  $ tcf power-on TARGETNAME
  $ tcf debug-info TARGETNAME
  GDB server: tcp:myhostname:3744 (when target is on; currently ON)

at this point, the target is waiting for the debugger to connect
before powering up, so start (in this case) GDB pointing it to the
*elf* version of the image file uploaded and issue::

  gdb> target remote tcp:myhostname:3744

Some targets might support starting debugging after power up.

Find more:

- `tcf --help | grep debug-`

- the :class:`debug extension API <tcfl.target_ext_debug.debug>` to
  :class:`target objects <tcfl.tc.target_c>` for use inside Python testcases

- the :class:`*ttbd*\'s Debug interface <ttbl.tt_debug_mixin>`

Zephyr debugging with TCF run
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When using targets in a high usage environment, it is easier to use
TCF run and a few switches:

- Make sure the target is acquired for a max of 30min while you work with it::

    $ for ((count = 0; count < 240; count++)); do tcf -t 1234 acquire TARGETNAME; sleep 15s; done &

  be sure to kill this process when done, to free it for other people;
  every 15s this re-acquires, as a way to tell the daemon you are
  still using it, to not free it from you.

  Note `-t 1234`; this says *use ticket 1234* to reserve this target;
  we'll use it later.

- Create a temporary directory::

    $ mkdir tmp

- Build and deploy::

    $ tcf -t 1234 run -vvvv -E --tmpdir tmp -t TARGETNAME --no-release PATH-TO-TC

  note the following:

  - `-t 1234` says to use ticket 1234, as the one we used for the reservation

  - `-E` tells it not to evaluate -- it will just build and deploy / flash

  - `--no-release` says do not release the target when done (because
    you want to do other stuff, like debug)

  - In the case of Zephyr, the ELF file will be in
    `tmp/1234/outdir-1234-SOMETHING/zephyr/zephyr.elf`, which you can
    find with::

      $ find tmp/1234/ -iname zephyr.elf
      tmp/1234/outdir-1234-j38h-quark_d2000_crb/zephyr/zephyr.elf

- Tell the target to start debugging::

    $ tcf -t 1234 debug-start TARGETNAME

- Now reset / power cycle it, so it goes fresh to start. Because we
  told it to start debugging, it will start but stop the CPU until you
  attach a debugger (only for OpenOCD targets or targets that support
  debugging, anyway)::

    $ tcf -t 1234 reset TARGETNAME
    $ tcf -t 1234 debug-info TARGETNAME
    OpenOCD telnet server: srrsotc03.iind.intel.com 20944
    GDB server: x86: tcp:srrsotc03.iind.intel.com:20946
    Debugging available as target is ON

- Note we now can start the debugger; find it first::

    $ find /opt/zephyr-sdk-0.9.3/ -iname \*gdb
    ...
    /opt/zephyr-sdk-0.9.3/sysroots/x86_64-pokysdk-linux/usr/bin/i586-zephyr-elf/i586-zephyr-elf-gdb
    ...

  run the debugger to the ELF file we found above::

    $ /opt/zephyr-sdk-0.9.3/sysroots/x86_64-pokysdk-linux/usr/bin/i586-zephyr-elf/i586-zephyr-elf-gdb \
        tmp/1234/outdir-1234-j38h-quark_d2000_crb/zephyr/zephyr.elf
    ...
    Reading symbols from tmp/1234/outdir-1234-j38h-quark_d2000_crb/zephyr/zephyr.elf...done.

  tell the debugger to connect to the GDB server found by running
  `debug-info` before::

    (gdb) target remote tcp:srrsotc03.iind.intel.com:20946
    Remote debugging using tcp:srrsotc03.iind.intel.com:20946
    0x0000fff0 in ?? ()

  Debug away!::

    (gdb) b _main
    Breakpoint 1 at 0x180f71: file /home/inaky/z/kernel.git/kernel/init.c, line 182.
    (gdb) c
    Continuing.
    target running
    redirect to PM, tapstatus=0x08302c1c
    hit software breakpoint at 0x00180f71

    Breakpoint 1, _main (unused1=0x0, unused2=0x0, unused3=unused3@entry=0x0) at /home/inaky/z/kernel.git/kernel/init.c:182
    182	{
    (gdb)
    ...

- on a separate terminal, you can:

  - read the target's console output with::

      $ tcf console-read --follow TARGETNAME

  - issue CPU resets, halts, resumes or OpenOCD commands (for targets
    that support it)::

      $ tcf debug-reset TARGETNAME
      $ tcf debug-halt TARGETNAME
      $ tcf debug-resume TARGETNAME
      $ tcf debug-openocd TARGETNAME OPENOCDCOMMAND

Note that resetting or power-cycling the board will create a new GDB
target with a different port, so you will have to reconnect that GDB
wo the new target remote reported by *tcf debug-info*.
      
How can I quickly build and deploy an image to a target?
--------------------------------------------------------

You can use the boilerplate testcase :download:`test_zephyr_boots.py
<../examples/test_zephyr_boots.py>`, which will build any Zephyr app
and try to boot it and see if it prints the Zephyr boot banner::

  $ ZEPHYR_APP=path/to/source tcf run -t TARGETNAME /usr/share/tcf/examples/test_zephyr_boots.py
  $ tcf acquire TARGENAME
  $ <work on it> ...

TCF will build your code configuring it properly for the chosen target
and deploy it. You want to inmediately acquire so it is not
powered-off by the daemon.


TCF's `run` says something failed, can I get more detail?
---------------------------------------------------------

FIXME: update

TCF's ``run`` tries to be quiet to the console, so when you run a lot
of tests on a lot of targets, the forest lets you see the trees.

When you need more detail, you can:

- add `-v`\s after ``run`` (but then it gives you detail everywhere)

- log to a file with `--log-file=FILENAME` and when something fails,
  grep for it::

    FAIL0/kaoe ../Makefile[quark]@.../frankie: (dynamic) build failed
    PASS1/tctp ../Makefile[x86]@.../mv-03: (dynamic) build passed

  that build failed; take those four letters next to the ``FAIL0``
  message (`kaoe`) -- that's a unique identifier for each message, and
  look for it with `grep`, printing 30 lines of context before the match::

    $ grep -B 100 kaoe FILENAME
    ....
    FAIL3/iigx ../Makefile[quark]@.../frankie: @build failed [2] ('make -j -C samples/hello_world/nanokernel BOARD=quark_se_sss_ctb  O=outdir-httpsSERVER5000ttb-v0targetsfrankie-quark_se_sss_ctb-quark_se_sss_ctb' from /home/inaky/z/kernel.git/samples/.tcdefaults:48)
    FAIL3/iigx ../Makefile[quark]@.../frankie: output: FF make: Entering directory '/home/inaky/z/kernel.git/samples/hello_world/nanokernel'
    FAIL3/iigx ../Makefile[quark]@.../frankie: output: FF make[1]: Entering directory '/home/inaky/z/kernel.git'
    ...

  most likely, the complete failure message will be right before the
  final failure message -- and you can now tell what happened. In this
  case, there is no good configuration for the chosen target

  The output driver can be changed to lay out the information
  diferently; look at :class:`tcfl.report.report_c`.

.. _linux_c_c:

How do I send Ctrl-C to a target?
---------------------------------

Trick over the serial console is that it is a pure pipe, there is no
special characters. So a quick way to do it is::

  $ tcf console-write TARGETNAME $(echo -e \\x03)

where that *\\x03* is the hex code of Ctrl-C. *man ascii* can tell you
the quick shortcuts for others.

How do I send a Linux command to a Linux target's console
---------------------------------------------------------

Try::

  $ tcf console-write TARGETNAME "ping 192.168.1.1"

Note that once the command is sent, the console, for whatever the
target cares, is still connected, even if the *console-write* command
returned for you. :ref:`Sending a Ctrl-C <linux_c_c>` is not as in a
usual Linux console.

.. _manual_install:

Manual installation of TCF from source
======================================

Creation and setup of user *ttbd*
---------------------------------

The *ttbd* user is used to store TCF's software and files, and the
*ttbd* group to give normal user access to said files.

To create it (if not yet created)::

  # useradd -G kvm ttbd

Allow members of group *ttbd* access to *ttbd*\'s home so they can
write files needed for the deployment in there; never shall need to
login as the *ttbd* user::

  # chmod g+ws ~ttbd

Allow other users access to *ttbd*\'s files::

  # usermod -aG ttbd USER1 USER2 ….  # Make USERs members of ttbdgroup

Install required software
-------------------------

Run::

  # dnf install -y openocd make gcc-c++ python-ldap pyserial \
    python-requests git python-werkzeug python-tornado python-flask \
    python-flask-login python-flask-principal pyusb python-pexpect \
    pyOpenSSL

to install software packages required by TCF's server:

- openocd is used to interact with targets
- gcc-c++ and make are used to build
- Git is a source control manager we’ll use to obtain TCF’s source
- python-ldap, pyserial and python-requests are Python libraries TCF relies on
- python-werkzeug, python-flask* and python-tornado are the HTTP
  server framework TTBD uses to serve data.
- pyusb is the library used to access USB devices from Python
- python-pexpect is a expect-like language implementation in Python


You might need to :ref:`install support packages
<install_support_pkgs>` that are not in distributions dependning on
what you want to run with TCF.

Remove conflicting packages
---------------------------

Remove *Modem Manager*, as it interferes with the serial ports::

  # dnf remove -y ModemManager

this won't be needed if you will only use QEMU devices.

Install TCF
-----------

Obtain the code

.. include:: 04-HOWTOs-LL-install-tcf.rst

.. _tcf_install_manual:

**Install the TCF client**::

  $ cd tcf.git
  $ python setup.py install --user

.. note:: you will also need to install the `Zephyr SDK 0.9
          <https://www.zephyrproject.org/downloads/tools>`_ to
          ``/opt/zephyr-sdk-0.9.3`` if you want to build Zephyr OS
          apps and other dependencies:

          Fedora::

            # dnf install -y make python-requests python-ply cmake \
              PyYAML python2-junit_xml python2-jinja2

          Ubuntu::

            # apt-get install -y python-ply python-requests make \
              python-junit.xml python-jinja2

**Install the server**

Install requirements: sdnotify, a library to help TCF’s server TTBD
integrate with systemd::

  $ sudo pip install sdnotify

Now the server itself::

  $ cd tcf.git/ttbd
  $ sudo python setup.py install

Note the server needs configuration of SELinux, kernel credentials,
UIDs and GIDs for operation -- so running it off the RPM package can
get complicated.

.. _install_support_pkgs:

Manual installation of support packages
=======================================

These are dependencies needed when certain kind of test hardware is
going to be connected or certain OSes / testcases are to be used:

- Arduino Due boards: :ref:`Bossa command line <bossac>`
- Zephyr SDK for Arduino 101, Intel Quark, FRDM, SAM e70, others:
  :ref:`Zephyr SDK <zephyr_sdk>`
- :ref:`ESP32 <xtensa_esp32>`
- NiosII on Altera MAX10: :data:`Quartus programmer
  <ttbl.tt.tt_max10.quartus_path>` and :data:`NiosII CPU image
  <ttbl.tt.tt_max10.input_sof>`

.. _bossac:

Bossac: Arduino Due flasher
---------------------------

If Arduino Due is going to be used, a special tool for flashing has to
be installed--this has to be built from source as the branch that
supports the Arduino Due hasn't been merged into mainline as of
writing these instructions:

1. Get the requiements and the code::

     # sudo dnf install -y gcc-c++ wxGTK-devel
     $ git clone https://github.com/shumatech/BOSSA.git bossac.git
     $ cd bossac.git
     $ git checkout -f 1.6.1-arduino-19-gae08c63

2. Build and install::

     $ make bin/bossac
     $ sudo install -o root -g root -m 0755 bin/bossac /usr/local/bin

3. (optional) Create a fast RPM with :ref:`FPM <fpm>`::

     $ chmod 0755 bin/bossac
     $ fpm  -n bossac -v $(git describe --tags) -s dir -t rpm bin/bossac

.. _zephyr_sdk:

Zephyr SDK
----------

To build some Zephyr OS apps/testcases or to flash certain hardware,
you will need this SDK:

1. Download the Zephyr SDK from
   https://www.zephyrproject.org/downloads/tools

2. Install in */opt/zephyr-sdk-VERSION*::

     # chmod a+x zephyr-sdk-0.9.3-setup.run
     # ./zephyr-sdk-0.9.3-setup.run -- -y -d /opt/zephyr-sdk-0.9.3

3. (optional) Create a fast RPM with :ref:`FPM <fpm>`::

     $ fpm -n zephyr-sdk-0.9.3 -v 0.9.3 \
     >     --rpm-rpmbuild-define '_build_id_links alldebug' \
     >     -s dir -C / -t rpm opt/zephyr-sdk-0.9.3

   *_build_id_links alldebug* is needed to disable generation of build
   symlinks in */usr/lib/.build-id*. Because the SDK packs a lot of
   files that are similar/identical to those present in the system, it
   will conflict.

.. _arduino_builder:

Arduino Builder 1.6.13
----------------------

The Arduino Builder will be needed by the TCF client to build *.ino*
files into appplications that can be flashed into targets for test.


1. Download the Arduino IDE package from
   https://www.arduino.cc/download_handler.php?f=/arduino-1.6.13-linux64.tar.xz

2. Extract::

   # tar xf arduino-1.6.13-linux64.tar.xz -C /opt

3. (optional) Create a fast RPM with :ref:`FPM <fpm>`::

   $ fpm -n tcf-arduino-builder-1.6.13 -v 1.6.13 -s dir -C / -t rpm opt/arduino-1.6.13/


tunslip6
--------

*tunslip6* is used to create a SLIP interface for a QEMU virtual
machine and connecting it to a TAP interface. This code has been
floating around for different small OSes that also run in QEMU, so
there is a few versions.

The one we currently use is the one used by the Zephyr project at the
*net-tools* repository:

  http://github.com/zephyrproject-rtos/net-tools

which has added functionality to defer configuration to external
parties (*ttbd* in this case) and some strenghtening to deal with race
conditions.


1. Clone::

     $ git clone http://github.com/zephyrproject-rtos/net-tools

2. Build::

     $ cd net-tools
     $ make tunslip6
     # make install

3. (optional) Create a fast RPM with :ref:`FPM <fpm>`::

     $ install -m 0755 tunslip6 -D root/usr/bin/tunslip6
     $ fpm -n tunslip6 -v $(git describe --always) -s dir -C root -t rpm usr/bin

.. _xtensa_esp32:

Xtensa ESP32
------------

In order to build (TCF client) and deploy/flash (ttbd server) for
ESP32 boards, you will need the *xtensa-esp32* SDK and the ESP-IDF
libraries:

- *xtensa-esp32 SDK*

  1. Download from
     https://dl.espressif.com/dl/xtensa-esp32-elf-linux64-1.22.0-59.tar.gz

  2. Extract to */opt/xtensa-esp32-elf*

     # tar xf xtensa-esp32-elf-linux64-1.22.0-59.tar.gz -C /opt

  3. Add to */etc/environment*::

       ESPRESSIF_TOOLCHAIN_PATH=/opt/xtensa-esp32-elf

- *ESP-IDF*: This is Xtensa's IOT framework that is used by Zephyr and others

  1. Clone to */opt/esp-idf.git*::

       $ rm -rf /opt/esp-idf.git
       $ git clone --recursive https://github.com/espressif/esp-idf.git /opt/esp-idf.git
       $ (cd /opt/esp-idf.git && git checkout -f $(ESP_IDF_REV))

  2. Add to */etc/environment*::

       ESP_IDF_PATH=/opt/esp-idf.git


libcoap
-------

Use the following script to create the RPM::

.. code-block:: sh

   # Use absolute dirs, libtool needs it
   dir=$PWD/libcoap.git
   rootdir=$PWD/root-libcoap.git

   rm -rf $dir
   git clone --recursive -b dtls https://github.com/obgm/libcoap.git $dir

   rm -rf $rootdir
   mkdir -p $rootdir

   (cd $dir; ./autogen.sh)
   sed -i 's|prefix = @prefix@|prefix = $(DESTDIR)/@prefix@|g' $(find $dir/ext/tinydtls -iname Makefile.in)
   (cd $dir; ./configure --disable-shared --disable-documentation --prefix=/usr)
   make -C $dir all
   make -C $dir DESTDIR=$rootdir install
   rm -rf $rootdir/usr/share
   ver=${ver:-$(git -C $dir describe --tags)}
   fpm  -n libcoap -v $ver -s dir -t rpm -C $rootdir

Invoke as::

  $ ./mklibcoap.sh


.. _fpm:

FPM, fast package manager
-------------------------

.. note:: this is only optional and only needed if you are building
          RPMs for distribution; please ignore otherwise

1. Install dependencies, clone the code, build and install::

     $ sudo dnf install -y ruby-devel rpm-build
     $ git clone https://github.com/jordansissel/fpm fpm.git
     $ make -C fpm.git install

Platform firmware / BIOS update procedures
==========================================

.. _a101_fw_upgrade:

Updating the firmware in an Arduino101 to factory settings
----------------------------------------------------------

This is needed to operate with Zephyr OS > v1.5.0-349-gecf96d2, which
dropped support for the older Arduino101 Zephyr boot ROM.

- Download from https://software.intel.com/en-us/node/675552 the
  package ``arduino101-factory_recovery-flashpack.tar.bz2`` and
  decompress to your home directory::

    $ cd
    $ tar xf LOCATION/arduino101-factory_recovery-flashpack.tar.bz2

- Ensure your TTBD server is >= v0.10 (dated 9/21/16 or later)

- Disable you Arduino101 target (to avoid it being used by other
  automated runs) and acquire it::

    $ tcf disable arduino101-NN
    $ tcf acquire arduino101-NN

- Flash the new Boot ROM and bootloader::

    $ tcf images-upload-set arduino101-NN \
      rom:$HOME/arduino101-factory_recovery-flashpack/images/firmware/FSRom.bin \
      bootloader:$HOME/arduino101-factory_recovery-flashpack/images/firmware/bootloader_quark.bin

- Release the target and enable it::

    $ tcf release arduino101-NN
    $ tcf enable arduino101-NN

- Test with::

    $ cd LOCATION/OF/ZEPHYR/KERNEL
    $ export ZEPHYR_BASE=$PWD
    $ tcf run -v -t arduino101-NN samples/hello_world


Updating the firmware in an Quark C1000 reference boards
--------------------------------------------------------

This updates to the QSMI v1.3 bootrom support, needed to operate with
Zephyr OS > v1.5.0, which dropps support older ROMs.

- Download from https://github.com/quark-mcu/qmsi/releases/tag/v1.3.0 the
  file ``quark_se_rom-v1.3.0.bin`` to your home directory::

    $ wget https://github.com/quark-mcu/qmsi/releases/download/v1.3.0/quark_se_rom.bin \
        -O ~/quark_se_rom-v1.3.0.bin

- Ensure your TTBD server is >= v0.10 (dated 11/01/16 or later)

- Disable you Quark C1000 target (to avoid it being used by other
  automated runs), maybe wait for no still-running jobs are using it
  (use `tcf list -v qc1000-NN` to see if it is acquired by anyone)::

    $ tcf disable qc1000-NN

For the actual flashing process, there is a list of steps that have to
be performed that are needed to wipe the flash and reset the board in
such a way that it puts it in a receptive state. This implies running
specific OpenOCD commands and modifying the way the OpenOCD driver
operates on the board to maintain those settings.

1. Use the script `tcf-qc1000-fw-upload.sh TARGETNAME FWFILE` to
   update the Boot ROM::

     $ tcf-qc1000-fw-upload.sh qc1000-NN $HOME/quark_se_rom.bin

   In case of failure, retry one or two times, as some commands get
   stuck. If after the retries it still fails, it might be time to try
   remediation steps as described below.

2. Verify operation with running a Zephyr test case::

     $ cd LOCATION/OF/ZEPHYR/KERNEL
     $ export ZEPHYR_BASE=$PWD
     $ tcf run -vat qc1000-NN  /usr/share/tcf/examples/test_healtcheck.py

   Testcase should `PASS`. Note the `-a`, so TCF can use a disabled
   target. If they fail and `tcf console-read qc1000-NN` reports
   garbage instead of some ASCII text, it is possible board timings
   are messed up. Go back to flashing the Boot ROM and maybe use the
   remediation steps described below.

3. Re-enable the target::

    $ tcf enable qc1000-NN

The process is thus concluced

Remediation steps
^^^^^^^^^^^^^^^^^

In some situations it has been seen that the stock OpenOCD version
distributed with the Zephyr SDK or most system cannot flash properly
the QC1000.

In said case, we can try with the ISSM version of OpenOCD:

1. obtain the ISSM toolchain package for Linux from
   https://software.intel.com/en-us/articles/issm-toolchain-only-download

2. Install in your system in path opt with::

     # tar xf PATH/TO/issm-toolchain-linux-2016-05-12-pub.tar.gz  -C /opt

3. Disable ISSM's OpenOCD using a TCL port, otherwise the TCF server,
   TTBD, will not be able to talk to it::

     # sed -i 's/tcl_port/#tcl_port/' \
       /opt/issm-toolchain-linux-2016-05-12/tools/debugger/openocd/scripts/board/quark*.cfg

4. Alter the TCF's configuration of each QC1000 target to be updated
   so it uses the OpenOCD from ISSM; in file
   `/etc/ttbd-production/conf_FILE.py`:

   .. code-block:: python

      quark_c1000_add(
          "TARGETNAME",
          serial_number = "SERIALNUMBER",
          ykush_serial = "YKUSHHUBSERIALNUMBER",
          ykush_port_board = YKUSHHUBPORT,
          openocd_path = "/opt/issm-toolchain-linux-2016-05-12/tools/debugger/openocd/bin/openocd",
          openocd_scripts = "/opt/issm-toolchain-linux-2016-05-12/tools/debugger/openocd/scripts")

5. Restart the server::

     # systemctl restart ttbd@production

6. Run a healthcheck on the target::

     $ tcf run -vat TARGETNAME  /usr/share/tcf/examples/test_healtcheck.py

   In case of trouble, diagnose by looking at the :ref:`journal
   <systemd_tips_diagnosis>`::

     $ journalctl -aeu ttbd@production

6. Retry the firmware update script

7. Upon success, revert the configuration change and restart the server

.. _fw_update_d2000:

Updating the firmware in an Quark D2000 reference boards
--------------------------------------------------------

This is needed to operate with Zephyr OS.

- Download from
  https://github.com/quark-mcu/qm-bootloader/releases/tag/v1.3.0 the
  file ``quark_d2000_rom.bin`` to your home directory::

    $ wget https://github.com/quark-mcu/qm-bootloader/releases/download/v1.4.0/quark_d2000_rom_fm_hmac.bin

- Ensure your TTBD server is >= v0.10 (dated 9/21/16 or later)

- Disable you Quark D2000 target (to avoid it being used by other
  automated runs) and acquire it::

    $ tcf disable qd2000-NN
    $ tcf acquire qd2000-NN

- Flash the new Boot ROM and bootloader::

    $ tcf images-upload-set qd2000-NN rom:quark_d2000_rom_fm_hmac.bin

- Release the target::

    $ tcf release qd2000-NN

- Verify operation with running a Zephyr test case::

     $ export ZEPHYR_BASE=LOCATION/OF/ZEPHYR/KERNEL
     $ tcf run -vat qd2000-NN  /usr/share/tcf/examples/test_healtcheck.py

   Testcase should `PASS`. Note the `-a`, so TCF can use a disabled
   target. If they fail and `tcf console-read qc1000-NN` reports
   garbage instead of some ASCII text, it is possible board timings
   are messed up. Go back to flashing the Boot ROM and maybe use the
   remediation steps described below.

- Enable the target::

    $ tcf enable qd2000-NN

Updating the FPGA image in Synopsys EMSK boards
-----------------------------------------------

1. You will need a Synopsys account; `register
   <https://www.synopsys.com/cgi-bin/dwarcsw/req1.cgi>`_ and wait for
   them to accept you

2. Use this account to `download their newest firmware
   <https://www.synopsys.com/cgi-bin/dwarcsw/arcemsk/menu.cgi>`_. Currently
   set to v2.2.

3. Now download the `Lab Tools 14.7` or newer (You may have to create
   a Xilinx account `here
   <https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/design-tools.html>`_
   to access this download)

   If you need installation instruction you can find them here under
   Appendix: C (page 86) of
   https://www.embarc.org/pdf/ARC_EM_Starter_Kit_UserGuide.pdf
   (important note, when running this it seems you have to use Windows
   and the Impact 32 bit version. The 64 bit version seems to fail out
   on certain steps you need to use).

4. Now just go through the flashing instructions located on page 93
   towards the bottom, called SPI Flash-Programming Sequence
   https://www.embarc.org/pdf/ARC_EM_Starter_Kit_UserGuide.pdf

   If you need another resource the synopsys instructions can be found
   here under (page 86) Appendix: C for installing the tools, and
   (page 93 bottom) under SPI Flash-Programming Sequence.
   https://www.embarc.org/pdf/ARC_EM_Starter_Kit_UserGuide.pdf

.. note::
   the switch configuration we are currently using is ARC_EM9D for the 9D model.

   Here is the switch configuration for SW1 (bit 1 is switch one; bit 2 is switch 2)

   ===== ==== =============
   Bit 1 Bit2 Configuration
   ----- ---- -------------
   OFF   OFF  ARC_EM7D
   ON    OFF  ARC_EM9D
   OFF   ON   ARC_EM11D
   ON    ON   Reserved
   ===== ==== =============

.. _fs2_serial_update:

Updating the serial number and / or description of a FTDI / Flyswatter
----------------------------------------------------------------------

FlySwatter2 come all with the same serial number (usually *FS20000*).

The system needs to be able to tell all the different FlySwatter2s
apart; for that we update the serial to *TARGETNAME-fs2* (to simplify
the configuration process).

Flash a new serial number or description on any FTDI based USB-to-TTY
cable/adapter (like a Flyswatter2, Quark D2000 CRB, Quark C10000 CRB,
etc) using ``ftdi_eeprom`` on your laptop::

  $ sudo dnf install -y libftdi-devel
  $ cat > file.conf <<EOF
  vendor_id=0x0403
  product_id=0x6010
  product="Flyswatter2"
  serial="NEWSERIALNUMBER"
  use_serial=true
  EOF

Now plug the USB cable to your server or laptop, making sure it is the
only one and run, as super user::

  # ftdi_eeprom --flash-eeprom file.conf

Reconnect it to have the system read the new serial number / description.

.. note:: if you have the unit you are re-flashing connected to a USB
   power switching hub (like a YKush), make sure to power it on and to
   power off any other device that has a 0x0403/6010 USB vendor ID /
   product ID code, which you can find with::

     $ lsusb.py  | grep 0403:6010
       1-1.2.1      0403:6010 00  2.00  480MBit/s 0mA 2IFs (Acme Inc. Flyswatter2 Flyswatter2-galileo-04)

   it might be easier to do this process in a separate system.

.. note:: make *NEWSERIALNUMBER* shorter if you receive this error
   message::

     FTDI eeprom generator v0.17(c) Intra2net AG and the libftdi developers <opensource@intra2net.com (opensource%40intra2net.com)>
     FTDI read eeprom: 0
     EEPROM size: 128
     Sorry, the eeprom can only contain 128 bytes (100 bytes for your strings).
     You need to short your string by: -1 bytes
     FTDI close: 0

.. admonition:: Rationales

   The *product="Flyswatter2"* line is necessary so the default
   OpenOCD configuration finds the device.

.. _cp210x_serial_update:

Updating the serial number and / or description of a CP210x serial device
-------------------------------------------------------------------------

Some hardware use serial-to-USB converters based on the CP210x series
by `Silicon Labs <http://www.silabs.com/>`_.

If the serial number programed on it is not unique enough, it can be
programmed with the tool ``cp120x-program``, available from
http://cp210x-program.sourceforge.net/.

Once installed:

1. identify the device to operate with::

     $ lsusb | grep -i CP210x
     Bus 002 Device 085: ID 10c4:ea60 Cygnal Integrated Products, Inc. CP210x UART Bridge / myAVR mySmartUSB light

   .. note:: your device might show differnt vendor or product ID and
             strings, in such case, adjust your grepping.

2. take the bus and device numbers (`002/085` in the example) and feed
   it to the `cp219x-program` tool with the new serial number you
   want::

     # cp210x-program -m 002/085 -w --set-serial-number "NEWSERIALNUMBER"

   it is always a good idea to set as new serial number the name of
   the target it is going to be assigned to.

3. Reconnect the device and verify the new serial number is set with
   ``lsusb.py``::

     $ lsusb.py -ciu
     ...
       2-2.4          10c4:ea60 00  1.10   12MBit/s 100mA 1IF  (Silicon Labs CP2102 USB to UART Bridge Controller esp32-39)
         2-2.4:1.0      (IF) ff:00:00 2EPs (Vendor Specific Class) cp210x ttyUSB0
     ..

   in this case, we set *esp32-39* as new serial number, which is
   displayed at the end of the line.
