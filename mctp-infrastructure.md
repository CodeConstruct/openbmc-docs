# MCTP infrastructure overview

Jeremy Kerr `<jk@codeconstruct.com.au>`

2021-08-23 - draft

This document gives a background of the MCTP infrastructure in OpenBMC, as a
general informative guide for development. This is not a formal OpenBMC
design document; those are at: [designs/mctp/mctp.md](designs/mctp/mctp.md).

# MCTP kernel core stack

The MCTP core code has been accepted into the `net-next` tree, but we need a few
follow-up patches for complete hardware support. Those are in the Code Construct
git tree: <https://github.com/CodeConstruct/linux> in the `dev/mctp` branch.
This contains some additions to the core, plus I²C transport support.

The MCTP network stack will need to be enabled through the kernel configuration
options:

  - `CONFIG_MCTP`

  - `CONFIG_MCTP_TRANSPORT_I2C`

A kernel built with these options will have support for MCTP sockets, and the
I²C driver included. However, this does not instantiate physical interface
devices - the hardware-specific configuration of MCTP transport interfaces is
covered in [Hardware interfaces: I²C/SMBus](#kernel-hw-ifaces) below.

## Userspace tools: `mctp` command

The `mctp` command provides a facility for low-level control of the MCTP network
stack, in a similar way to [iproute2](https://github.com/shemminger/iproute2)'s
`ip` command. This allows control of the local MCTP interfaces, addresses,
routes and neighbour tables. The MCTP userspace tools are available in the Code
Construct mctp git tree: <https://github.com/CodeConstruct/mctp>

As a brief overview:

`mctp link` can be used to query or update the local MCTP interfaces. With no
arguments, it prints all interfaces present on the system:

``` sh-session
# mctp link
dev lo index 1 address 0x00:00:00:00:00:00 net 1 mtu 65536 up
```

In the above listing, we have a single interface, `lo`, with a zero hardware
address (as `lo` is a loopback-type interface, it has no hardware address). This
link is on network id 1, and has a maximum transmit unit (MTU) size of 65536.
The link is currently up.

`mctp addr` can be used to query or update the addresses (MCTP endpoint IDs, or
EIDs) assigned to local interfaces. New addresses can be assigned through an
invocation like:

``` sh-session
# mctp addr add 8 dev lo
```

\- this will assign the EID of 8 to the `lo` interface. Interfaces may have
multiple local addresses.

With no arguments, `mctp addr` will print all addresses on the system:

``` sh-session
# mctp addr
eid 8 net 1 dev lo
```

## Hardware interfaces: I²C/SMBus

A device-tree binding specifies which I²C controllers may host MCTP endpoints.
Adding a mctp-i2c node to a top-level I²C controller will signify that this
controller hosts MCTP devices, and will result in a MCTP interface being created
for that controller. The `reg` property of this node specifies the I²C/SMBus
client address used for the local MCTP interface.

<div class="note">
Note:
The device tree bindings for MCTP devices are currently being discussed with the
upstream community. This section describes the bindings as currently
implemented; this *may* change in response to future feedback.

</div>

As an example, we add an mctp interface on the `i2c6` controller, with I²C/SMBus
client address of `0x10`:

``` dts
&i2c6 {
     status = "okay";

     mctp@10 {
         compatible = "mctp-i2c";
         reg = <(0x10 | I2C_OWN_SLAVE_ADDRESS)>;
     };
};
```

The `I2C_OWN_SLAVE_ADDRESS` flag indicates that this SMBus address is for a
local SMBus client endpoint - the client address is assigned to the local
machine rather than describing a remote I²C device. This constant is defined in
a I²C-specific device tree binding header; if your top-level device tree doesn't
already include it, add:

``` c
#include <dt-bindings/i2c/i2c.h>
```

to define the value.

With the sample device tree configuration shown above, we see a new MCTP
interface, named `mctpi2c6` (reflecting the underlying I²C bus name of `i2c6`):

``` sh-session
# mctp link show
dev lo index 1 address 0x00:00:00:00:00:00 net 1 mtu 65536 up
dev mctpi2c6 index 4 address 0x10 net 1 mtu 254 down
```

The `address 0x10` output indicates that this device is using local SMBus
address of `0x10`, as defined by the `reg` property in the device tree.

With this interface present, we can then assign a local EID:

``` sh-session
# mctp addr add 8 dev mctpi2c6
# mctp addr show
eid 8 net 1 dev mctpi2c6
```

... and bring the interface up:

``` sh-session
# mctp link set mctpi2c6 up
# mctp link show mctpi2c6
dev mctpi2c6 index 4 address 0x10 net 1 mtu 254 up
```

At this point, we have MCTP connectivity to any remote MCTP devices. However, we
need to be able to discover those remote devices - the new `mctpd`
infrastructure provides facilities to discover and configure remote endpoints.

# mctpd

The mctp userspace tools repository also includes sources for an MCTP daemon
(`mctpd`), an implementation of the MCTP Control Protocol. This is used for
enumerating external MCTP endpoints, assigning addresses, managing the local
MCTP route- and neighbour-tables. It also exposes data about discovered MCTP
endpoints to other applications, through a lightweight dbus interface.

A systemd.service file is shipped with the mctp sources, under the `conf/`
directory, enabling `mctpd` to be managed by systemd.

<div class="note">
Note:
Any local interface and address setup should be performed before `mctpd`
is started. This may be implemented as a custom systemd init script calling
`mctp addr add` and/or `mctp link set`.
</div>

Once started, `mctpd` will claim the `xyz.openbmc_project.MCTP` dbus bus name,
and expose base-level objects for MCTP control:

    # busctl tree xyz.openbmc_project.MCTP
    └─/xyz
      └─/xyz/openbmc_project
        └─/xyz/openbmc_project/mctp 
          └─/xyz/openbmc_project/mctp/1
            └─/xyz/openbmc_project/mctp/1/8 

There are two objects of interest here:

  - `/xyz/openbmc_project/mctp`: The control interface for mctpd
  - `/xyz/openbmc_project/mctp/1/8`: An endpoint object; in this case,
    representing the local EID

These objects are described in the following two sections.

## mctpd's control interface

The first of `mctpd`'s dbus interfaces is the server control interface. This is
located at the dbus path `/xyz/openbmc_project/mctp`, and implements the
au.com.CodeConstruct.MCTP interface:

``` sh-session
# busctl introspect xyz.openbmc_project.MCTP /xyz/openbmc_project/mctp
NAME                                TYPE      SIGNATURE  RESULT/VALUE  FLAGS
au.com.CodeConstruct.MCTP           interface -          -             -
.AssignEndpoint                     method    say        yisb          -
.LearnEndpoint                      method    say        yisb          -
.SetupEndpoint                      method    say        yisb          -
```

<div class="note">
Note:
The au.com.CodeConstruct namespace is only temporary, while the interfaces
are in development; once these interfaces have been standardised, they will
likely move to OpenBMC-specific names.
</div>

These three dbus calls (`AssignEndpoint`, `LearnEndpoint` and `SetupEndpoint`)
provide facilities for applications to trigger enumeration of new hardware
endpoints. All three take the same set of arguments to specify the new endpoint
to discover:

 * `yisb AssignEndpoint(s:: interface, ay:: physaddr)`
 * `yisb LearnEndpoint(s:: interface, ay:: physaddr)`
 * `yisb SetupEndpoint(s::interface, ay:: physaddr)`

On a successful probe of the specified address, all three return the following
data:

  - `y (byte)`:: EID  
    The endpoint ID of the discovered device

  - `i (integer)`:: network ID  
    The network ID of the discovered device

  - `s (string)`:: path  
    The dbus path of the new device, referring to the MCTP.Endpoint object.
    
    <div class="note">
    Note:
    The `path` return value will likely change type in a future patch - from
    string (s) to object-path (o). However, this will have no impact on
    serialisation format.
    </div>

  - `b (boolean)`:: new  
    A flag indicating whether the device was newly-discovered or assigned a new
    EID; semantics differ slightly between calls, see below for a full
    description.

The most appropriate function to use will depend on the state and pre-existing
configuration of remote endpoints. The behaviour of each functions is as
follows:

  - `AssignEndpoint`  
    Will send a *Set Endpoint ID* control protocol message to that physical
    address, assigning it a new EID. The boolean `new` return value is set to
    true when a new EID is assigned.
    
    This method is intended for cases where the remote endpoint is known not to
    have an EID, and needs one assigned.

  - `LearnEndpoint`  
    Queries with a *Get Endpoint ID* control protocol message to determine the
    existing EID of an endpoint. The boolean `found` return value is set if a
    EID is assigned. This may be false if the device responds but does not have
    an EID assigned yet.
    
    This method is intended for cases where the remote endpoint is known to have
    an EID, and the device just needs to be enumerated.

  - `SetupEndpoint`  
    First performs a `LearnEndpoint` on the device, and if no EID is set it will
    perform `AssignEndpoint`. Returns a true `new` return value if a new EID was
    assigned to the device. `SetupEndpoint` is the usual method to be called by
    libraries to configure devices.
    
    This method is intended for general device initialisation; the device may or
    may not have an EID assigned, and mctpd will detect the appropriate
    initialisation and enumeration actions to perform.

The top-level mctpd object (at `/xyz/openbmc_project/mctp`) also implements the
org.freedesktop.DBus.ObjectManager interface, so the list of enumerated MCTP
endpoints is retrievable through the `GetManagedObjects` method call, without
needing to know the network and endpoint IDs of each. Individual objects are
described in the next section.

## mctpd's endpoint enumeration interface

Results of `mctpd` enumeration are also represented as dbus objects, using the
OpenBMC-specified MCTP endpoint format. Each endpoint appears on the bus at the
object path:

    /xyz/openbmc_project/mctp/<network-id>/<endpoint-id>

and exposes three dbus interfaces for each:

  - xyz.openbmc\_project.MCTP.Endpoint  
    Provides MCTP address information (`EID` and `NetworkID` properties) and
    message-type support (`SupportedMessageTypes` property).
    
    This interface is defined by the
    [MCTP.Endpoint](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/MCTP/Endpoint.interface.yaml)
    phosphor-dbus specification.

  - xyz.openbmc\_project.Common.UUID  
    MCTP UUID of the discovered endpoint (`UUID` property).
    
    This interface is defined by the
    [Common.UUID](https://github.com/openbmc/phosphor-dbus-interfaces/blob/master/yaml/xyz/openbmc_project/Common/UUID.interface.yaml)
    phosphor-dbus specification.

  - au.com.CodeConstruct.MCTP.EndPoint  
    Additional control methods for the endpoint - for example, `Remove`

We can trigger device enumeration using the known physical address of the
device, and calling the `SetupEndpoint` on the mctpd control interface
(described in the [mctpd's control interface](#mctp-iface-control) above).

As an example, we can call `SetupEndpoint` to probe an NVMe device that is
connected to interface `mctpi2c6`, and uses the NVMe-MI default SMBus address of
`0x1d`:

<div class="informalexample">

    # busctl call xyz.openbmc_project.MCTP \
            /xyz/openbmc_project/mctp \
            au.com.CodeConstruct.MCTP \
            SetupEndpoint say mctpi2c6 1 0x1d
    yisb 9 1 "/xyz/openbmc_project/mctp/1/9" true

</div>

After issuing this `SetupEndpoint` call, we now see the new device appear in the
enumerated device object set:

<div class="informalexample">

    # busctl tree xyz.openbmc_project.MCTP
    └─/xyz
      └─/xyz/openbmc_project
        └─/xyz/openbmc_project/mctp
          └─/xyz/openbmc_project/mctp/1
            ├─/xyz/openbmc_project/mctp/1/8
            └─/xyz/openbmc_project/mctp/1/9

</div>

And can introspect the new object (output slightly abbreviated):

<div class="informalexample">

``` sh-session
# busctl introspect xyz.openbmc_project.MCTP /xyz/openbmc_project/mctp/1/9
NAME                                TYPE      SIGNATURE RESULT/VALUE
au.com.CodeConstruct.MCTP.Endpoint  interface
.Remove                             method    -         -           
xyz.openbmc_project.Common.UUID     interface
.UUID                               property  s         "54663803-0000-[…]"
xyz.openbmc_project.MCTP.Endpoint   interface
.EID                                property  y         9           
.NetworkId                          property  i         1           
.SupportedMessageTypes              property  ay        1 4         
```

</div>

The addressing data (network ID and EID) allows us to communicate with the MCTP
device using the kernel sockets interface.

`mctpd` is also responsible for maintaining the kernel routing and neighbour
tables. This provides the necessary local configuration to allow userspace
applications to exchange messages with remote endpoints, addressed by their EID
alone. After the enumeration actions described above, we can see that the route
table now contains an entry to reach our NVMe device on EID 9:

    # mctp route
    eid min 9 max 9 net 1 dev mctpi2c6 mtu 0
    eid min 8 max 8 net 1 dev mctpi2c6 mtu 0

This entry specifies that the device with EID 9 can be reached through the
`mctpi2c6` interface.

We can also query the neighbour table, to find an entry for our NVMe device:

    # mctp neigh
    eid 9 net 1 dev mctpi2c6 lladdr 0x1d

This provides the mapping of the MCTP EID (of 9) to the physical I²C/SMBus
address `0x1d`.

# Integration

## Base OpenBMC integration

### systemd units & targets

For a consistency between BMC platforms, we propose two new targets for platform
MCTP infrastructure initialisation:

  - `mctp-local.target`  
    Target to achieve local stack initialisation: fixed interfaces are
    initialised, and have been assigned addresses.

  - `mctp.target`  
    MCTP infrastructure (ie., `mctpd`) is running, and dbus interfaces are
    active.

Services that require an MCTP transport to be active (eg, PLDM requesters and
responders) should depend on `mctp.target`, and not `mctp-local.target`. The
former indicates that general MCTP facilities are available, the latter
indicates that the local hardware is configured.

<div class="note">
Note:
These two systemd services are designed to support the current static
initialisation patterns. There may be future changes required for hotplug
support for local MCTP interfaces, but the general pattern of using
`mctp.service` for MCTP applications will be preserved.

</div>

#### System-specific device configuration using `mctp-local.target`

The `mctp-local` target depends on the local MCTP stack initialisation; it has
no dependencies by default, but machine-specific configuration can be
implemented by new units that add their own dependency, typically through a
`WantedBy` option. For example:

<div class="informalexample">

``` ini
[Unit]
Description=MCTP configuration for i2c bus 6
Requires=sys-subsystem-net-devices-mctpi2c6.device

[Service]
Type=oneshot
ExecStart=/usr/bin/mctp link set mctpi2c6 up
ExecStart=/usr/bin/mctp addr add 8 dev mctpi2c6
RemainAfterExit=true

[Install]
Wantedby=mctp-local.target
```

</div>

Systems with multiple MCTP interfaces may be better serviced through a template
service file, with the MCTP device provided as an instance name.

The mctp userspace tools repository includes a set of base configuration files
intended to assist integration into a systemd-based environment:

  - `conf/mctp-local.target`  
    Systemd target definition for local hardware initialisation

  - `conf/mctp.target`  
    Systemd target definition for application-level MCTP support

  - `conf/mctpd-dbus.conf`  
    dbus permission configuration for the mctpd dbus interface service
    
    Can be installed to `/usr/share/dbus-1/system.d/mctpd.conf` on the BMC
    system.

  - `conf/mctpd.service`  
    Systemd service unit definition to start `mctpd` on boot
    
    Can be installed to `/usr/lib/systemd/system/mctpd.service` on the BMC
    system.

A yocto recipe for mctpd will be developed soon. As further integration parts
are developed, this document will be updated accordingly.
