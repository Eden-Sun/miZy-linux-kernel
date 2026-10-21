# XR819 stability review — 2026-10-21

The pinned driver, `3ce657ae533e2213f6fda2297da1598fa11d7a03`, is the latest
merged commit in fifteenhex/xradio as checked through GitHub API. Armbian's
current drivers_network.sh pins the same commit. Its last merged change adds
management frame protection capability (PR #29, 2026-09-01).

The tested H2+ remains connected after 37 minutes, but its kernel log repeatedly
reports `xradio_wlan ... missed interrupt`. Successful boot and HTTPS requests
are functional smoke tests, not evidence of long-term reliability.

## Unmerged candidates

- [#31](https://github.com/fifteenhex/xradio/pull/31): preserve the data cipher
  when rejecting an unsupported management key. The submitter reports malformed
  ARP replies with WPA2/PMF. The diff moves cipherType assignment after cipher
  validation. Potentially relevant to transient ARP reachability; not diagnosed
  as the cause on this board.
- [#34](https://github.com/fifteenhex/xradio/pull/34): correct HT receive metadata
  written to the wrong field. Its submitter still reports packet loss in 802.11n
  and uses 802.11g. This is a report, not a measured result on this board.
- [#32](https://github.com/fifteenhex/xradio/pull/32): honor lower transmit power
  limits instead of always requesting 20 dBm.
- [#33](https://github.com/fifteenhex/xradio/pull/33): broad bounds, buffer and
  firmware-error handling changes intended to reduce lockups. The author says
  it needs testing; it is not an established stable release.
- [#30](https://github.com/fifteenhex/xradio/pull/30): experimental firmware/driver
  stability changes; its author says the branch was not hardware-tested.

None of these unmerged changes has been installed on the tested board. Keep the
working image and backup as the baseline. A useful next experiment is reviewing
and testing #31 and #34 separately in RAM, measuring packet loss, reconnects,
missed interrupts and sustained HTTPS traffic before considering persistence.
Do not claim that disabling 802.11n or applying these patches fixes this board
without measuring it.

Sources:
- https://github.com/fifteenhex/xradio/commits/master/
- https://github.com/armbian/build/blob/main/lib/functions/compilation/patch/drivers_network.sh
