
This is how I run automatic upgrades on archlinux.

Debian has [unattended-upgrades](https://wiki.debian.org/UnattendedUpgrades) which works well. This does the same, for arch.

## Install

```
sudo pacman -S --needed base-devel git
git clone https://github.com/kousu/arch-automatic-upgrades.git
cd arch-automatic-upgrades/
(cd arch-automatic-upgrades && makepkg -fsri)
```

To also get unattended updates for AUR packages, install [pikaur](https://aur.archlinux.org/packages/pikaur):

```
(cd $(mktemp -d) && git clone https://aur.archlinux.org/pikaur.git && cd pikaur && makepkg -fsri)
(cd arch-automatic-upgrades-aur && makepkg -fsri)
```

## Set up notifications

When run on a server, you definitely want to keep an archive of updates.

I use opensmtpd, but you could use postfix (or sendmail if you're masochistic).

```
sudo pacman -S opensmtpd s-nail
sudo systemctl enable --now smtpd
```

Test:

```
echo hello | mail -s "testing" you@example.com
```

```
echo you@example.com | sudo tee -a ~root/.forward
```


Test again:

```
echo hello | mail -s "testing" root
```

This should get forwarded to you@example.com, and that's good because that means messages from cron will be too.


This is a bit trickier to get right. You need to futz with your DNS to make this work: I like https://mail-tester.com/ to help me get this right.
The most important thing is that the forward and reverse DNS addresses exist and match.
You can add SPF records to help them out too. You probably don't need DKIM or DMARC.
On most mail servers these days, just marking a few notifications not-spam will be enough for them to learn to trust your server.

# Thoughts

This will install updates once a week on Sunday, sending you a report about any config files you need to investigate.

Arch has in some circles a reputation for being [too unstable for automatic upgrades](https://wiki.archlinux.org/index.php/User:Andy_Crowd/Update_packages_from_crontab). Not so!
I've been running this on server and laptops for literally years now and it has almost never failed me,
except when a package grew too large to be built on the tiny server I had,
or [when dealing with postgres](https://wiki.archlinux.org/index.php/PostgreSQL#Upgrading_PostgreSQL) (I still haven't figured out how to automate that; there's a reason it's not already automated).

You can also skip setting up notifications. In that case, you will have to run `sudo mail` to read
the upgrade logs and find the messed up config files.

You should strongly consider using this in tandem with `etckeeper`: just `sudo pacman -S etckeeper` and all updates and mods will be logged.
