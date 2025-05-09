# Use the Fedora minimal image from registry.fedoraproject.org
FROM registry.fedoraproject.org/fedora:latest

# Update package lists and install dnf5 (if not already present)
RUN dnf -y install dnf5 dnf5-plugins procps-ng libX11-common && \
    dnf clean all

RUN \
    # Create PlexMedia yum repository
    dnf5 config-manager addrepo \
    --set=baseurl='https://downloads.plex.tv/repo/rpm/$basearch/' \
    --set=gpgkey=https://downloads.plex.tv/plex-keys/PlexSign.key \
    --id=PlexRepo \
    --set=enabled=1 \
    && \
    \
    dnf clean all \
    \
    # Add group 'plex' if not already present on system.
    if [ $(cat /etc/group | grep ^plex | wc -l) -eq 0 ]; then \
        groupadd plex \
    fi \
    && \
    \ 
    # Add user 'plex' if not already present on system.
    if [ $(cat /etc/passwd | grep ^plex: | wc -l) -eq 0 ]; then \
        useradd -d /var/lib/plexmediaserver -c "RPM Created PlexUser" -g plex --system -s /sbin/nologin plex \
    fi \
    && \
    \
    # Add user 'plex' to 'video' group if not added already.
    # Presume User and Group to be 'plex' and 'video'.
    PUSER=plex \
    VGROUP=video \
    NORUN=0 \
    # If the default group for video devices isn't 'video', switch to that group.
    if [ -f /dev/dri/renderD128 ]; then \
        if [ "$(stat --format=%G /dev/dri/renderD128)" != "root" ]; then \
            VGROUP="$(stat --format=%G /dev/dri/renderD128)" \
        else \
            NORUN=1 \
        fi \
    elif [ -f /dev/fb0 ]; then \
        if [ "$(stat --format=%G /dev/fb0)" != "root" ]; then \
            VGROUP="$(stat --format=%G /dev/fb0)" \
        else \
            NORUN=1 \
        fi \
    elif [ -f /dev/video0 ]; then \
        if [ "$(stat --format=%G /dev/video0)" != "root" ]; then \
            VGROUP="$(stat --format=%G /dev/video0)" \
        else \
            NORUN=1 \
        fi \
    fi \
    && \
    \
    if [ "${NORUN}" != "1" ]; then \
        if [ "$(getent group ${VGROUP}|grep -c ${VGROUP})" != "0" ]; then \
            if [ "$(groups ${PUSER}|grep -c ${VGROUP})" = "0" ]; then \
                usermod -a -G ${VGROUP} ${PUSER} \
            fi \
        fi \
    fi \
    && \
    \
    # Install Plex
    dnf5 download plexmediaserver \
    rpm -i --noscripts plexmediaserver*rpm \
    && \
    \
    # Postinstall tasks
    # Create default PMS library directory for user 'plex' if it doesn't exist.
    if [ ! -d /var/lib/plexmediaserver ]; then \
        mkdir /var/lib/plexmediaserver \
        chown plex:plex /var/lib/plexmediaserver \
    fi \
    && \
    \
    # Add PMS SELinux policy file if SELinux is present and not disabled.
    if [[ -f /usr/sbin/getenforce && ! $(/usr/sbin/getenforce | grep -i Disabled) ]]; then \
        semodule -i /usr/lib/plexmediaserver/plexrsync.pp \
    fi \
    && \
    \
    # If udev is installed, check for tv butler rules file and create if needed.
    if [[ -d /lib/udev/rules.d && ! -f /lib/udev/rules.d/60-tv-butler-perms.rules ]]; then \
        echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="1d19", ATTRS{idProduct}=="0100", GROUP="video", MODE="0664"' > /lib/udev/rules.d/60-tv-butler-perms.rules \
    fi \
    && \
    \
    # Setup directories
    mkdir -p \
        /config \
        /transcode \
        /data \
    && \
    \
    # Run Plex
    /bin/sh -c '/usr/bin/test -d "${PLEX_MEDIA_SERVER_APPLICATION_SUPPORT_DIR}" || /bin/mkdir -p "${PLEX_MEDIA_SERVER_APPLICATION_SUPPORT_DIR}"' \
    /bin/sh -c '\
    PLEX_MEDIA_SERVER_INFO_VENDOR="$(grep ^NAME= /etc/os-release | awk -F= "{print \\$2}" | tr -d \\" )" \
    PLEX_MEDIA_SERVER_INFO_MODEL="$(uname -m)" \
    PLEX_MEDIA_SERVER_INFO_PLATFORM_VERSION="$(grep ^VERSION= /etc/os-release | awk -F= "{print \\$2}" | tr -d \\" )" \
    "/usr/lib/plexmediaserver/Plex Media Server"'

# Install and Start Plex
#COPY runplex .
#CMD ["/bin/bash","-c","./runplex"]

EXPOSE 32400/tcp 8324/tcp 32469/tcp 1900/udp 32410/udp 32412/udp 32413/udp 32414/udp
VOLUME /config /transcode /media

ENV CHANGE_CONFIG_DIR_OWNERSHIP="true" \
    HOME="/config"

#COPY root/ /

RUN chgrp -R 0 /run /usr/local /var/cache /var/log /var/run \
    && chmod -R g=u /run /usr/local /var/cache /var/log /var/run
    
#HEALTHCHECK --interval=5s --timeout=2s --retries=20 CMD /healthcheck.sh || exit 1

# Create a directory for persistent data and set permissions
RUN mkdir /data && \
    chown daemon /data && \
    chmod 755 /data

# Switch to the 'daemon' user (non-interactive shell)
USER daemon

# Set the working directory
WORKDIR /
