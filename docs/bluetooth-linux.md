# Bluetooth adapters on Linux

OBDium talks to ELM327 adapters through a serial port. A Bluetooth adapter has
to be exposed as a serial device first. How depends on which kind of Bluetooth
the adapter speaks:

| Adapter type | Serial device | Section |
|---|---|---|
| Classic Bluetooth (SPP), including dual-mode adapters | `/dev/rfcomm0` | [Classic adapters](#classic-adapters-rfcomm) |
| Bluetooth Low Energy only | `/tmp/ttyBLE` | [BLE-only adapters](#ble-only-adapters) |

Many adapters sold as "BLE" are dual-mode and also offer the classic serial
profile. Check before assuming (step 2 below); the classic route is simpler.

## Before you start

- Plug the adapter into the vehicle. It is powered by the OBD-II port.
- Disconnect it from any phone. An adapter accepts one connection at a time
  and may stop advertising while a phone holds it.
- Keep the computer within a few metres of the vehicle.
- Your user must be able to open serial devices:
  `sudo usermod -aG dialout $USER`, then log out and back in.
- The Bluetooth radio must be on:
  `sudo rfkill unblock bluetooth && bluetoothctl power on`.

## Classic adapters (rfcomm)

### 1. Find the adapter's address

```sh
bluetoothctl --timeout 15 scan on
bluetoothctl devices
```

Look for a name such as `OBDII`, `VEEPEAK`, `Vgate` or `ELM327` and note the
address (`AA:BB:CC:DD:EE:FF`).

### 2. Check that it offers a serial port, and on which channel

```sh
sdptool browse AA:BB:CC:DD:EE:FF
```

A classic adapter lists a `Serial Port` service with an `RFCOMM` channel,
usually `1`. If there is no such service, the adapter is BLE-only; go to
[BLE-only adapters](#ble-only-adapters).

### 3. Pair only if the adapter asks for it

Many adapters accept a connection without pairing. Skip this step and come
back to it if step 4 fails with a permission or authentication error.

```sh
bluetoothctl pair AA:BB:CC:DD:EE:FF    # PIN is usually 1234 or 0000
bluetoothctl trust AA:BB:CC:DD:EE:FF
```

### 4. Bind it to a serial device

```sh
sudo rfcomm bind 0 AA:BB:CC:DD:EE:FF 1
```

The arguments are the device number (`0` gives `/dev/rfcomm0`), the address
and the channel from step 2. Binding does not connect; the link is opened
when a program opens the device, so the adapter does not need to be in range
yet.

### 5. Connect in OBDium

Turn the ignition on, start OBDium, open the **Connection** panel and click
refresh. Port discovery probes every serial port and can take 10 to 15
seconds. Select `/dev/rfcomm0` and connect.

### Keeping the binding across reboots

A manual `rfcomm bind` is lost on reboot. The repository ships a systemd
template unit, [`contrib/linux/rfcomm-bind@.service`](../contrib/linux/rfcomm-bind@.service),
that restores it at boot. The instance name is the rfcomm device number.

```sh
sudo install -m 644 contrib/linux/rfcomm-bind@.service /etc/systemd/system/
sudo mkdir -p /etc/rfcomm-bind
printf 'ADDRESS=AA:BB:CC:DD:EE:FF\nCHANNEL=1\n' | sudo tee /etc/rfcomm-bind/0.conf
sudo systemctl daemon-reload
sudo systemctl enable --now rfcomm-bind@0.service
```

For a second adapter, write `/etc/rfcomm-bind/1.conf` and enable
`rfcomm-bind@1.service`; it appears as `/dev/rfcomm1`.

To change adapters, edit the `.conf` file and run
`sudo systemctl restart rfcomm-bind@0.service`. To remove the binding, run
`sudo systemctl disable --now rfcomm-bind@0.service`.

## BLE-only adapters

A BLE-only adapter has no serial profile. A bridge such as
[`ble-serial`](https://github.com/Jakeler/ble-serial) turns its GATT
characteristics into a pseudo-terminal.

```sh
pipx install ble-serial
ble-scan                          # find the adapter's address
ble-scan -d AA:BB:CC:DD:EE:FF     # list its services and characteristics
ble-serial -d AA:BB:CC:DD:EE:FF   # creates /tmp/ttyBLE; leave it running
```

Do not pair the adapter in `bluetoothctl`; the bridge connects on its own. If
`ble-serial` picks the wrong characteristics, pass the notify and write UUIDs
from the scan with `-r` and `-w`.

OBDium probes `/tmp/ttyBLE` automatically when it exists. A pseudo-terminal
at another path is not listed by the operating system, so name it in
`OBDIUM_EXTRA_PORTS` (a `:`-separated list of paths):

```sh
OBDIUM_EXTRA_PORTS=/tmp/my-adapter obdium
```

## Troubleshooting

Start `obdium` from a terminal to see what port discovery finds. Each probe
prints a line ``- response: `...` for port: ...`` and the result ends with
`Ports: [...]`.

| Symptom | Likely cause |
|---|---|
| Adapter does not show up in a scan | A phone is still connected to it, it is out of range, or the radio is blocked (`rfkill list`). |
| `br-connection-profile-unavailable` from a BLE tool | The adapter is dual-mode and the system prefers the classic side. Use the rfcomm route. |
| `/dev/rfcomm0` is missing from the port list | The list has not finished loading (wait 15 seconds), the adapter did not answer (range, phone connected), or your user is not in `dialout`. |
| Port is listed but connecting fails | Check the terminal for `error opening serial port`. Make sure no other program holds the device. |
| A sensor shows "unsupported" | The vehicle's ECU does not report that PID. This is independent of the adapter. |
