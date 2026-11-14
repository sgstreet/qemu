.. _input-linux:

Linux evdev input passthrough and the INPUT_LINUX_GRAB event
=============================================================

.. note::

   The ``INPUT_LINUX_GRAB`` event described here is a local addition to
   the ``input-linux-grab-event-v10.2`` branch (based on QEMU 10.2.1). It
   is not part of upstream QEMU.

The ``input-linux`` object passes a host evdev device (keyboard, mouse)
to the guest. While the device is *grabbed*, host input is delivered only
to the guest. A configurable key combination toggles the grab, which
makes it possible to share one keyboard and mouse between the host and a
guest, like a KVM switch.

Whenever an ``input-linux`` object successfully grabs or releases its
device, QEMU emits the ``INPUT_LINUX_GRAB`` QMP event. Management tools
can use it to react to the change (for example to switch a monitor input)
without QEMU spawning any process.

Command line
------------

.. code-block:: shell

   qemu-system-x86_64 ... \
     -object input-linux,id=kbd1,evdev=/dev/input/by-id/<kbd>-event-kbd,grab_all=on,repeat=on,grab-toggle=ctrl-ctrl \
     -object input-linux,id=mouse1,evdev=/dev/input/by-id/<mouse>-event-mouse

``evdev``
  Path of the host device. Use the stable ``/dev/input/by-id/`` names,
  not ``eventN``.

``grab_all``
  If ``on``, toggling this device also toggles every ``input-linux``
  object that does not set ``grab_all`` itself. Set it on the keyboard
  only, so the mouse follows the keyboard.

``repeat``
  Enable auto-repeat events.

``grab-toggle``
  The key combination that toggles the grab: ``ctrl-ctrl`` (default),
  ``alt-alt``, ``shift-shift``, ``meta-meta``, ``scrolllock`` or
  ``ctrl-scrolllock``.

The user running QEMU needs read/write access to the evdev node. On most
distributions that means membership of the ``input`` group.

The INPUT_LINUX_GRAB event
--------------------------

Data:

``id`` (string)
  The id of the ``input-linux`` object (``kbd1`` above).

``active`` (bool)
  ``true`` if the device is now grabbed (input goes to the guest),
  ``false`` if it was released (input goes to the host).

Example::

   { "event": "INPUT_LINUX_GRAB",
     "data": { "id": "kbd1", "active": false },
     "timestamp": { "seconds": 1763100000, "microseconds": 123456 } }

Behaviour to be aware of:

* The event is sent only after the ``EVIOCGRAB`` ioctl succeeded. A
  failed grab or release produces no event.
* With ``grab_all=on`` one key press toggles several objects. Each
  object that changes state sends its own event, with its own ``id``.
  Filter on the keyboard's id to get one notification per key press.
* The grab taken when QEMU starts happens before a QMP client can
  connect, so that first ``active=true`` is not seen. Query or set the
  initial state yourself when the guest starts.
* No event is sent when the object goes away (guest shutdown, or
  ``object-del``). The kernel drops the grab when QEMU closes the
  device, but nothing reports it. React to ``SHUTDOWN`` or to libvirt's
  lifecycle events if you need a "released" state.

Quick test without libvirt::

   qemu-system-x86_64 -display none -machine none -nodefaults \
     -qmp unix:/tmp/qmp.sock,server,nowait \
     -object input-linux,id=kbd1,evdev=/dev/input/by-id/<kbd>-event-kbd,grab_all=on,repeat=on,grab-toggle=ctrl-ctrl
   (echo '{"execute":"qmp_capabilities"}'; cat) | socat - UNIX-CONNECT:/tmp/qmp.sock

Press both Ctrl keys. ``active`` alternates between ``false`` and
``true``. Note that the keyboard is grabbed while this runs.

Using the event with libvirt
----------------------------

libvirt passes QMP events it has no native handler for through as
"QEMU monitor events", so no libvirt change is needed.

Domain XML
~~~~~~~~~~

Select the QEMU build in the domain and add an evdev input. Give the
keyboard a user alias so its object id is stable (libvirt otherwise
numbers inputs by their position in the list, ``input0``, ``input1``,
...). The alias must start with ``ua-``::

   <devices>
     <emulator>/usr/local/bin/qemu-system-x86_64</emulator>
     <input type="evdev">
       <source dev="/dev/input/by-id/<mouse>-event-mouse"/>
     </input>
     <input type="evdev">
       <source dev="/dev/input/by-id/<kbd>-event-kbd" grab="all" repeat="on"/>
       <alias name="ua-kbd"/>
     </input>
     ...

libvirt probes the emulator, so the build must contain every feature the
domain uses. If ``virsh edit`` reports an error such as "USB redirection
is not supported by this version of QEMU", reconfigure QEMU with the
missing feature (for example ``--enable-usb-redir``, ``--enable-spice``,
``--enable-slirp``) and install the matching ``-dev`` packages first.

Check the id on the running guest::

   virsh -c qemu:///system dumpxml <domain> | grep -B2 -A2 evdev

Watching the event::

   stdbuf -oL virsh -c qemu:///system qemu-monitor-event \
       --event INPUT_LINUX_GRAB --loop --timestamp

``virsh`` buffers its output when it is piped, hence ``stdbuf -oL``.

AppArmor (Ubuntu)
~~~~~~~~~~~~~~~~~

Ubuntu confines libvirt and each guest with AppArmor. A QEMU installed
under ``/usr/local`` needs two local overrides, using the include hooks
Ubuntu already provides.

Allow libvirtd to run the binary::

   echo '/usr/local/bin/qemu-system-* PUx,' | \
       sudo tee -a /etc/apparmor.d/local/usr.sbin.libvirtd

Allow the confined guest to use the binary, its data files and the evdev
nodes::

   sudo mkdir -p /etc/apparmor.d/local/abstractions
   sudo tee /etc/apparmor.d/local/abstractions/libvirt-qemu <<'EOF'
   /usr/local/bin/qemu-system-* rmix,
   /usr/local/share/qemu/** rk,
   /dev/input/event* rw,
   EOF

Reload and restart::

   sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.libvirtd
   sudo systemctl restart libvirtd

libvirt's AppArmor helper does not know about evdev inputs, so the
``/dev/input`` rule is required. Narrow it to the specific ``eventN``
node if you prefer. If a guest fails to start, look for denials with
``journalctl -k | grep 'apparmor="DENIED"'``. If you use bridged
networking through ``qemu-bridge-helper`` in ``/usr/local/libexec``, it
needs its own rule.

Running a command when the grab changes (e.g. ddcutil)
-------------------------------------------------------

Libvirt hooks only run on lifecycle events, not on QMP events, and QEMU
must not spawn processes. Run a small watcher outside both, for example a
systemd user service. This example switches a monitor input with
``ddcutil`` (``active=true``: guest input, ``active=false``: host input).

``~/.local/bin/ilg-monitor-switch``::

   #!/usr/bin/env bash
   # Usage: ilg-monitor-switch <domain> <object-id>
   DOMAIN=${1:?}; ID=${2:?}
   DISPLAY_NO=1
   GUEST_INPUT=0x0f   # monitor input wired to the guest GPU
   HOST_INPUT=0x11    # monitor input wired to the host

   stdbuf -oL virsh -c qemu:///system qemu-monitor-event \
       --event INPUT_LINUX_GRAB --loop --timestamp |
   while read -r line; do
       [[ $line == *"for domain '$DOMAIN'"* &&
          $line == *"\"id\":\"$ID\""* ]] || continue
       if [[ $line == *'"active":true'* ]]; then
           val=$GUEST_INPUT
       else
           val=$HOST_INPUT
       fi
       ddcutil --display "$DISPLAY_NO" setvcp 60 "$val" --noverify &>/dev/null
   done

Find the input values with ``ddcutil detect`` and ``ddcutil getvcp 60``,
once with each input selected.

``~/.config/systemd/user/ilg-monitor-switch.service``::

   [Unit]
   Description=Switch monitor input on INPUT_LINUX_GRAB

   [Service]
   ExecStart=%h/.local/bin/ilg-monitor-switch <domain> ua-kbd
   Restart=always
   RestartSec=2

   [Install]
   WantedBy=default.target

Enable it::

   systemctl --user daemon-reload
   systemctl --user enable --now ilg-monitor-switch

The user needs to be in the ``libvirt`` group (for ``qemu:///system``)
and in the ``i2c`` group for ``ddcutil``.

Caveats:

* ``ddcutil`` talks to the monitor over the DDC wires of the GPU cable
  it uses. If the monitor is only connected to a GPU that is passed
  through to the guest, the host cannot reach it. Connect the monitor to
  a host GPU as well, and use a monitor that answers DDC on an inactive
  input.
* The initial grab and the release at shutdown produce no event (see
  above). Set the initial monitor input from a libvirt ``started`` hook
  (run ``ddcutil`` in the background) and restore the host input on
  ``stopped``.
* ``ddcutil`` can take a second or more per call, so the switch lags the
  key press slightly.
