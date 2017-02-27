# NoDDos - Stop DDos attacks at the source

The NoDDos client monitors network traffic in the home and dynamically applies device-specific ACLs to that traffic to stop a device from being used in a DDOS attack. The ACLs are downloaded from the cloud and are generated based on traffic data uploaded anonymously by the NoDDos client. You can install the NoDDos client on Linux DIY routers and on Home Gateways running OpenWRT. . For more information see the [NoDDos website](https://www.noddos.io/).

## Client Overview

The noddos client consists of the following tools:
- nodlisten.py: receives DNS, DHCP and SSDP information from the LAN and persists it in a local database
- getdeviceprofiles.sh: tool to download device profiles from the Noddos web site
- makecert.sh: create certificate used to perform client authentication for REST calls to the Noddos cloud
- nodreport.py: reads the local database and attempts to match them against the device profiles and then uploads traffic and/or host data to the cloud
- janitor.py: keeps the local database to a manageable size

##### Nodlisten.py
Nodlisten runs as a daemon to listen to DHCP, DNS and SSDP traffic on the home netowrk. It persists collect data to a SQLite3 database
. It reads DHCP and DNS data from the dnsmasq daemon that should be configured to log extended DNS and DHCP data. If incoming SSDP dat
a has a 'Location' header than nodlisten will call the URL contained in the header to collect additional device information. Configura
tion can be set in a configuration file (see below) that can be overridden by command line options. The process should be started at b
oot time.

##### getdeviceprofiles.sh
The 'getdeviceprofiles.sh' script is used to securely download the list of Device Profiles over HTTPS from the Noddos web site, check the digital signature of the file using a Noddos certificate and save the verified file under '/var/lib/noddos/DeviceProfiles.json' by default, although this can be modified with the --directory command line parameter. It needs access to the public cert that was used to sign the file by default from /etc/noddos/noddosconfig.crt but this can be modified with the --certificate parameter. This script should be called at least once per day from cron. This script does not read the noddosconfig.json configuration file (see below) so use the command line options if the defaults do not apply.

##### clientcert.sh

Creates certificate used to call the Noddos cloud REST APIs with. The certificate is used to limit the scope of DDOS attacks while removing the associating between uploaded data and the IP address of the client. See the section on privacy for more details.

##### nodreport.py

Nodreport should periodically (ie. every hour) be called from cron to match the data generated by nodlisten with the device profiles that were downloaded by getdeviceprofiles.sh. If a host matches a device profile than the traffic statistics are uploaded to the cloud. The traffic statistics consist of the DNS queries that the host sent together with the IP flow statistics collected by the ulogd2 daemon. If the host doesn't match a device profile then detailed DNS/DHCP/SSDP information about the host is uploaded to the cloud so that a device profile can potentially be created. The Nodos cloud does not store the source IP address of this data upload, maintaining the privacy of the user. For more details, please read the section on privacy.

##### janitor.py

The janitor tool should be called from cron to delete data from noddos.db to prevent it from growing to an unmanageable size.

## Installation

Pre-requisites
- Linux v2.6.13 or later (as inotify support is needed)
- python3
- dnsmasq
- (optional) ulogd2 ulogd2d-sqlite3 ulog2d-json
- sqlite3
- openssl
- wget (preferred because of conditional GET support) or curl
- gzip or bzip2 or brotli (latter is preferred due to superior compression rate)
- screen (optional)

NoDDos leverages ulogd2 data if it is available so on Linux routers running Ubuntu:

    sudo echo "1"> /proc/sys/net/netfilter/nf_conntrack_acct
    sudo echo "1"> /proc/sys/net/netfilter/nf_conntrack_timestamp
    sudo cat >/etc/sysctl.d/90-ulogd2.conf <<EOF
    net.netfilter.nf_conntrack_acct=1
    net.netfilter.nf_conntrack_timestamp=1
    EOF

Configure /etc/ulogd.conf for logging to SQLite3 DB

    # Enable / uncomment the SQLite3 plugin
    plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_SQLITE3.so"
    # Add a stack in the section immediately below the plugins
    stack=ct1:NFCT,noddos_sqlite3_ct:SQLITE3
    # Add at the end of the file:
    [noddos_sqlite3_ct]
    table="ulog_ct"
    db="/var/log/ulog/ulog.sqlite3db"
    buffer=200

Create Ulogd2 SQLite3 database

    sudo sqlite3 /var/log/ulog/ulog.sqlite3db <ulog.sql
    sudo chown ulog:adm /var/log/ulog/ulog.sqlite3db
    sudo chmod a+r /var/log/ulog/ulog.sqlite3db
    sudo service ulogd restart

NoDDos leverages dnsmasq logs. 

    cat >>/etc/dnsmasq.conf <<EOF
    log-queries=extra
    log-dhcp
    EOF

    sed -i 's|procd_set_param command $PROG -C $CONFIGFILE -k -x /var/run/dnsmasq/dnsmasq.pid|procd_set_param command $PROG -C $CONFIGFILE --log-facility /var/log/dnsmasq.log -k -x /var/run/dnsmasq/dnsmasq.pid|' /etc/init.d/dnsmasq

    cat >/etc/logrotate.d/dnsmasq <<EOF
    /var/log/dnsmasq.log
    {
        rotate 7
        daily
        su root syslog
        size 10M
        missingok
        notifempty
        #nodelaycompress
        compress
        postrotate
        /usr/bin/killall -SIGUSR2 dnsmasq
        endscript
    }
    EOF

Set up NoDDos 

    git clone https://github.com/noddos/noddos
    cd noddos

    sudo mkdir /var/log/noddos /etc/noddos/ /var/lib/noddos
    MYUSERNAME=`whoami`
    sudo chown $MYUSERNAME /var/log/noddos /etc/noddos /var/lib/noddos

    cp noddosconfig.crt /etc/noddos
    tools//getdeviceprofiles.sh 
    # Install a cronjob to do this frequently, ie
    # 0 */3 * * * /path/to/noddos/tools/getdeviceprofiles.sh

    cp noddosconfig-sample.json /etc/noddosconfig.json

    tools/makecert.sh
    mv noddosapiclient.pem /etc/noddos

    # Install cronjob to run nodreport.py every hour. Please chose a random
    # minute of the hour to help spread load to the cloud API
    # 38 * * * * /path/to/noddos/pyclient/nodreport.py

    # Nodlisten is still at alpha quality so best to run it from `screen'
    pyclient/listener/nodlisten.py --verbose debug --nodaemon

## Configuration
The noddos client configuration file (Default: /etc/noddos/noddosconfig.json, -c / --configurationfile command line parameter) is a JSON file with a JSON object under the 'client' key. Some of its settings can be overrriden by command line options. The keys for the configuration items in the JSON file are:

__whitelistmac__: list of ethernet MAC addresses that that should not have any data  uploaded to the cloud.
Default: empty list of strings
Command line option: none

__whitelistipv4__: list of IPv4 addresses that that should not have any data uploaded to the cloud.
Default: empty list of strings
Command line option: none

__whitelistipv6__: list of IPv6 addresses that that should not have any data uploaded to the cloud.
Default: empty list of strings
Command line option: none

__signaturecert__: certificate used to validate the digital signature for the DeviceProfiles.json file.
Default: /etc/noddos/noddosconfig.crt
Command line option: -c, --certificate (getdeviceprofiles.sh)

__clientcert__: authentication certificate for the client when calling NoDDos cloud APIs.
Default: /etc/noddos/noddosconfig.crt
Command line option: -e, --clientcertificate (nodreport.py)

__lastrunfile__: Keeps track of when nodreport was last executed. Hosts that are mapped to a device profile will only be re-evaluated by nodreport if t
he value of the LastUpdate field in the device profile is greater than the last time that nodreport ran.
Default: /var/log/noddos/reporter-lastrun.out
Command line option: none

__dbfile__: SQLite3 database that locally stores all information collected by nodlisten and the mappings from hosts to device profiles found by no
dreporter.
Default: /var/log/noddos/noddos.db
Command line option: -s, --dbfile (nodreport.py, nodlisten.py, janitor,py)

__ulogdbfile__: SQLite3 database that locally stores all information collected by ulogd.
Default: /var/log/ulog/ulog.sqlite3db
Command line option: -u, --ulogdbfile (nodreport.py, janitor,py)

__dnsmasqlog__: The dnsmasq daemon is configured per the installation instructions to write his extended DNS and DHCP logging to this file. Nodlisten
tails this file, parses the log lines and puts the data in noddos.db.
Default: /var/log/dnsmasq.log
Command line option: none
BUG: hardcoded in nodlisten because of some issues with calling inotify.

__deviceprofilesfile__: The list of deviceprofiles for matching hosts against.
Default: /var/lib/noddos/DeviceProfiles.json
Command line option: -p, --deviceprofiles (nodreport.py)

__apiserver__: The FQDN of the Noddos API server in the cloud.
Default: api.noddos.io
Command line option: -a, --apiserver (nodreport.py)

__logfilenodlisten__: File to which nodlisten sends logging info.
Default: (no logging)
Command line option: -l, --logfile (nodlisten.py)

__logfilenodreport__: File to which nodreport sends logging info
Default: (no logging).
Command line option: -l, --logfile (nodreport.py)

__loglevelnodlisten__: debug level of nodlisten, can be 'debug', 'info', 'warning', 'error' or 'critical'.
Default: warning
Command line option: -v, --verbose (nodlisten.py, nodreport.py)

__loglevelnodreport__: debug level of nodlisten, can be 'debug', 'info', 'warning', 'error' or 'critical'.
Default: warning
Command line option: -v, --verbose (nodreport.py)

__ipaddress__: IP address of the NIC connected to the home network for nodlisten to listen for SSDP traffic on.
Default: none
Command line option: -a --ip-address (nodlisten.py)

__interface__: BUG: not currently implemented. Interface on which nodlisten should listen for SSDP traffic. Mutually exclusive with 'ipaddress'.
Default: none
Command line option: -i, --interface (nodlisten.py)

__pidfile__: Location for pidfile of nodlisten daemon.
Default: /var/log/noddos/nodlisten.pid
Command line option: -p, --pidfile (nodlisten.py)

__nodaemon__: Nodlisten can remain in the foreground for debugging purposes if this is set to true.
Default: false
Command line option: -n, --nodaemon (nodlisten.py)

__expiredns__: The janitor will delete DNS queries older than the value for this key in seconds from noddos.db.
Default: 7 days
Command line option: -d, --dnsexpire (janitor.py)

__expiretraffic__: The janitor will delete IP flow stats older than the value for this key in seconds from noddos.db.
Default: 12 hours
Command line option: -t, --trafficexpire (janitor.py)

__expirehost__: The janitor will delete hosts not seen on the network for longer than the value for this key in seconds from noddos.db
Default: 7 days
Command line option: -e, --hostexpire (janitor.py)

## Installation on OpenWRT
The NoDDos client can be installed on OpenWRT but keep in mind that:
- There is no Ulogd2 support in OpenWRT so IP traffic stats will not be uploaded. SSDP, DHCP and DNS info will be uploaded so there is still a lot of value in running NoDDos on OpenWRT
- You'll need to install the full python3 package in OpenWRT.
- As there are a lot of writes to the SQLite3 database, you will want to have /var/log on a disk that is not using Flash storage
- You will need SSH access to your Home Gateway as there is no OpenWRT package for the NoDDos client
