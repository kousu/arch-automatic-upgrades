
This is how I run automatic upgrades on archlinux.

Maybe I'll figure out how to package this some day! If you know `makepkg` better than I do, lend me a hand?

Debian has [unattended-upgrades](https://wiki.debian.org/UnattendedUpgrades) which works well.
This does the same, for arch.

0. Dependencies.

    You probably already have these.

    ```
    sudo pacman -S sudo cronie # sudo to install sudo :P ?
    sudo systemctl enable --now cronie
    ```

2. Install `check-pacnew` from this repo:

    This script finds config (and other) files that both you and the package have changed.
    These files end up getting installed with a .pacnew extension, waiting for manual merging.
    This script finds them and diffs them so that you can eyeball what needs merging.

    ```
    curl -JLO https://raw.githubusercontent.com/kousu/arch-automatic-upgrades/master/sbin/check-pacnew
    chmod +x check-pacnew
    sudo mkdir -p /usr/local/bin
    sudo mv check-pacnew /usr/local/sbin
    ```

    Test:

    ```
    sudo check-pacnew
    ```
   
    It should run but at this first point give no output, though it might find some surprises if you've run your system for a while.


    ((what you *really* want to know is what patch to apply on top of the new version; in git terminology,
      you want to rebase your local patches on top of the package-maintainer's version.
      but we don't know the original version; maybe etckeeper can help?))
   
1. Install [pikaur](https://aur.archlinux.org/packages/pikaur) from the AUR; this is a bit of a manual process, but luckily you only need to do this once, because every other update, including pikaur itself, will be handled by pikaur:

    ```
    sudo pacman -S git pyalpm python-commonmark base-devel
    curl -JLO https://aur.archlinux.org/cgit/aur.git/snapshot/pikaur.tar.gz
    tar -zxvf pikaur.tar.gz
    cd pikaur*
    makepkg
    sudo pacman -U pikaur-*.tar.xz
    ```

1. Make a build user.

    When pikaur needs to build something from source it needs some space to work, and it wisely
    refuses to make a workspace with root permissions, so it should have its own dedicated user with dedicated storage where builds can be done.
    Its work ends up under `~automatic-upgrades/.local/share/pikaur/aur_repos/` and under `/tmp`

    ```
    useradd -m automatic-upgrades
    echo 'automatic-upgrades ALL=(ALL) NOPASSWD: /usr/bin/pacman' > /etc/sudoers.d/automatic-upgrades
    ```

3. Make things automatic:

    ```
    cat > /etc/cron.d/automatic-upgrades <<EOF
    @weekly		sudo -u automatic-upgrades pikaur -Syu --noconfirm && pacman -Sc --noconfirm && reboot
    # and since that's gonna leave unfinished merges around the system, try:
    0	0	*	*	Mon	/usr/local/sbin/check-pacnew 
    EOF
    ```

3. (optional) Update mirrors automatically too

    ```
    sudo pacman -S reflector
    sudo tee -a /etc/cron.d/ <<EOF
    15 17 */13 * * reflector -c <your-country> -f 3 > /etc/pacman.d/mirrorlist
    EOF
    ```
    
    **EDIT**: apparently it's now better to do
    
    ```
    sudo pacman -S reflector
    reflector --help # to learn the options
    sudo vi /etc/xdg/reflector/reflector.conf # configure to taste
    systemctl enable reflector.timer
    systemctl start reflector.timer
    ```

3. Set up notifications:

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
   

    This is a bit trickier to get right. You need to futz with your DNS to make this work:
   
    I like https://mail-tester.com/ to help me get this right.
    The most important thing is that the forward and reverse DNS addresses exist and match.
    You can add SPF records to help them out too. You probably don't need DKIM or DMARC.
    On most mail servers these days, just marking a few notifications not-spam will be enough
    for them to learn to trust your server.


# Thoughts

This will install updates once a week on Sunday, sending you a transcript of the install,
and on Monday it will email you a report about any config files you need to investigate.

Arch has in some circles a reputation for being [too unstable for automatic upgrades](https://wiki.archlinux.org/index.php/User:Andy_Crowd/Update_packages_from_crontab). Not so!
I've been running this on server and laptops for literally years now and it has almost never failed me,
except when a package grew too large to be built on the tiny server I had,
or [when dealing with postgres](https://wiki.archlinux.org/index.php/PostgreSQL#Upgrading_PostgreSQL) (I still haven't figured out how to automate that; there's a reason it's not already automated).

You can skip pikaur and the build user if you don't use the AUR:
  just replace 'pikaur' with 'pacman' and drop 'sudo -u automatic-upgrades'.
  But pikaur is a drop-in compatible with pacman and doing the extra steps means some day when
  you do want to use an AUR package you can.

You can also skip setting up notifications. In that case, you will have to run `sudo mail` to read
the upgrade logs and find the messed up config files.

You should strongly consider using this in tandem with `etckeeper`: just `sudo pacman -S etckeeper` and all updates and mods will be logged.
