# `arch-automatic-upgrades`

Unattended upgrades for ArchLinux.
Upgrades run once a week, at a point specific to the system
(see `systemctl list-timers`).

To also upgrade AUR packages automatically install [`arch-automatic-upgrades-aur`](../arch-automatic-upgrades-aur). Without it, they'll be left alone, but if an upgrade breaks one's dependencies the run fails (and doesn't reboot) so you can rebuild it manually.


## Schedule

To change the schedule:

```
# systemctl edit arch-automatic-upgrades.timer
```

Every other day (odd days of the month) at 7am:

```ini
[Timer]
# clears the previous schedule
OnCalendar=
OnCalendar=*-*-1/2 07:00
# no random delay
RandomizedDelaySec=0
```

Sundays at 15h:

```ini
[Timer]
OnCalendar=
OnCalendar=Sun *-*-* 15:00
RandomizedDelaySec=0
```

Keep `RandomizedDelaySec=0`, or the default's up-to-a-week delay still applies.

See systemd.timer(5), systemd.time(7), and use e.g. `systemd-analyze calendar --iterations=3 '*-*-1/2 07:00'` to check what an expression will do.

### Disabling

To disable the automatic part of the upgrades temporarily:

```
systemctl mask arch-automatic-upgrades.timer
```

## Reporting

If an upgrade leaves `.pacnew` files around or fails the details are mailed to `root`. To receive these mails, install an SMTP agent:

```
# pacman -S smtp-forwarder
```

Then forward them somewhere you actually read:

```
# echo you@example.com > ~/.forward
# chmod 644 ~root/.forward
```

Test it:

```
# printf 'Subject: test from %s\n\nhello\n' "$(hostname)" | sendmail root
```

## Recovering from failures

Most failures should be recoverable just by re-running

```
# arch-automatic-upgrades
```

When the upgrade comes out clean, reboot to ensure the updates go live.

## Outdated AUR Packages

A silent failure that can happen is if a library an AUR package
needs is upgraded underneath it. Then the AUR package will keep
trying to load the old library and failing.

This script tries to handle this automatically. But, if it fails for some reason
and your system is misbehaving, you can investigate if it's due to the AUR packages by running

```
# checkrebuild
```

and then trying to `pikaur -S --rebuild` each package listed, checking
their AUR pages for news, trying to build them manually with `makepkg` etc.
