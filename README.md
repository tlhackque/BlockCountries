BlockCountries
================

iptables manager to block by country

Copyright (C) 2010, 2012, 2013 Timothe Litt

BlockCountries generates iptables and ipv6tables 
that allow blocking IP traffic based on the country
that the IP address is assigned to by the issuing NIC.

This may or may not be where the traffic actually originates
(Tunnels, proxies, etc), but it ususally is.  It has proven
effective for me.

It supports both IPv4 and IPv6 addresses.  It is highly
configurable, allowing you to block a country but allow
access to some ports, or to only block all countries except
those that you wish to permit.  There are other options.

BlockCountries obtains the IP address data from the NICs.

The generated iptables are optimized for chain length,
generating subchains when it is advantageous.  

BlockCountries can (and should) be run as a cron job to get
updates to the IP address allocations, and as an initscript
at system startup.

If configured to log rejected connections, it will produce
some simple reports of the intercepts by host IP, showing the
country, protocols, ports and number of attempts.

It is a Perl script.

#Dependencies:
The script uses the following Perl library modules, available
from CPAN (or in the base Perl distribution).  Those preceeded
by # are only used if a particular command requires them.

````
File::Basename
File::Path
#IO::Uncompress::Gunzip
Locale::Country
#LWP::Simple
Net::IP;
NetAddr::IP
#Net::Domain
#Parse::Syslog
#POSIX
Regexp::IPv6 qw( $IPv6_re )
Socket
Storable
Text::ParseWords
````

#Cautions:

Consider carefully whether you want to use this software and the full consequences
to your site and\/or business.  By necessity, it will block potential customers and
'good' connections along with villains.  You must consider the costs and benefits
to your operation - the author does not endorse any specific policy.  In particular,
the defaults should be viewed as examples, not value judgements.

The IPv6 optimizations have not been tuned; it is not clear what the right
subchain structure is.

#Installation:
Copy BlockCountries to /etc/init.d (or your distributions startup method)
run chkconfig to include it in the autostart.

Inspect the first few lines and modify these variables for your environment:

````
$CFGFILE - This is where the configuration file lives

$ZONEDIR - This is where the files that define IP assignments live.  It should
           be a dedicated directory, must exist and must be writable by the
           cron job.
$LOG     - The syslog file containing iptables log entries.  Wildcard if
           logrotation occurs.

$LOGPFX  - The prefix to be written by IPtables when logging a rejection.
$LOGPGM  - The program to be credited with writing the log entry

The %config hash rarely needs adjustment; it defines where to find the iptables
commands, and the chain names to insert the filter into.  The path to the
commands is normally specified in the config file.

````

Add a crontab entry similar to:
````
CRONJOB=1
11 23 * * Sun /etc/init.d/BlockCountries start -update
````
Please pick a different time.

Create /etc/sysconfig/BlockCountries and put in your configuration.

Here is a sample (the countries are chosen to illustrate syntax, and do not
constitute a recommendation):

````
# Configuration for BlockCountries service

# Countries
#
# The list of contry codes and names can be obtained from BlockCountries with:
#
#     BlockCountries list
#
# This lists both ISO code and name for documentation (and as insurance against changes in the name)
# However, either would do.
lt Lithuania
md "Moldova, Republic of"

# Allow inbound mail, which requires DNS
#

-atport -atport smtp -atport submission -atport smtps -atport domain
-auport domain

# Filter both IPV4 and IPV6

-ipv4 -ipv6

# Enable logging

-log

# Path to iptables command

-path /usr/local/sbin

````

Run 
````
BlockCountries -start -update
````

The filter should download the latest IP country assignments and be in place.

````


#Usage:

The usage as of V2.0 follows (the words prefixed by $ are filled
in at runtime):

````
IP filter manager for country filters

Usage: $prog command args
  status [-v]        Display filter status.
                     -v provides configuration from config file
                     and command file - NOT iptables.
  list               List available country names/codes.
                     Contacts server for list.
  intercepts [-host name] [-days n]
                     List today\'s intercepts by host from $LOG.
                     Requires -log to start.
  stop  args         Stop filtering
  restart  args      Synonym for start (reloads with no open window)
  condrestart args   Restarts only if already running
  start args         Starts filter

Start uses tables of IP blocks assigned to country codes that are
stored in $ZONEDIR, which will be created if necessary.  The country
files are binary, and are only created for country codes that are
filtered.  The data is obtained from the Regional Internet Registries
when absent, or when start -update is specified.

iptables filters are generated and installed by start.  The filters are
optimized and generally will not look identical to the input data.  However
they will match the same address (no more and no fewer.)

Arguments for start-class commands are:
 -update            Get latest IP allocation data from regional registries.
                    Otherwise, only gets data if no local file exists for a registry.
 -4 -ipv4           Install filter for IPv4 addresses \ Default is -4 override with -no4 -no6
 -6 -ipv6           Install filter for IPv6 addresses / For both, use -4 -6
 -log               Install a logging rule to log rejected packets.
 -nolog             Don\'t install a logging rule. (default)
 -nolimit           Do not limit logging (can generate huge log files if under attack; not advised)
 -limit spec        Limit logging, default = $loglimits[0]:$loglimits[1] (see man iptables "limit")
 -atport n          Allow connections to TCP port n even FROM banned addresses.
                    May specify any number of times.  May use a service name.
 -auport n          Allow connections to UDP port n even FROM banned addresses.
                    May specify any number of times.  May use a service name.
 -atporto n         Same as -atport, but for connections TO banned addresses.
 -auporto n         Same as -auport, but for connections TO banned addresses.
 -aip ip(\/mask)    Allow connection from an otherwise banned IP address.
                    For a block, specify a netlength or mask. A hostname may
                    also be specified.  May specify any number of times.
 -dip ip(\/mask)    Deny connections from an otherwise allowed IP address.
                    Same syntax as -aip.
 -path /sbin        Path for iptables utilities (use in $CFGFILE)
 -permitonly        Listed countries will be permited, all others denied.
 -blockout          Also generate rules to block output & forwarded-output.
                    This is probably not required for most applications, and
                    will roughly double the memory requirements.
                    Caution: If you use -blockout for start, you must also use it for stop.
                    This will not be a problem if it\'s in the config file.
 -d                 Output random debugging messages
 -v                 Output extended status/statistics
  CC                ISO Country code or name to ban (or permit if -permitonly).
                    Specify as many as you like.

Arguments may also be obtained from $CFGFILE.
Anything (except comments) contained in it is prepended to every command
line\'s arguments.

Use single or double-quoted strings for country names containining spaces.

This script is designed to run as a service; chkconfig will link it into
/etc/rcn.d/.

This script should also be run with start -update from a cron job - weekly
is suggested - to obtain the latest IP address databases.  If the CRONJOB
environment variable is set, only errors and zone updates will be reported.
This is recomended to minimize e-mail from cron.

To prevent communications from banned IP addresses during updates,
start will install new rules before removing active rules.  For
this to be effective, you should not stop the service.

The status -v command will report the current configuration, although the
actual implementation in iptables will be different due to optimizations.

To view the actual iptables configuration, use
    $IPT4 -nvL --line-number | less for IPV4 or
    $IPT6 -nvL --line-number | less for IPV6

start -v will provide some statistics for the optimizations and generated rules.

The intercepts command will parse $LOG and summarize intercepts by IP address.
It will break down the dropped packets by protocol(s) and port(s).  Of course,
logging must be on for this to work.  -days specifys how many days back
(from the current time) to read.  -host specifies the hostname to match.
Default -days is 1, hostname is current host.

Credits:
 Some ideas came from http://www.cyberciti.biz/faq/block-entier-country-using-iptables/.

 This version of the script merges all the IP address blocks; this saves over 1,000
 rules for the default banned address list.  It\'s also somewhat faster than a shell
 script, and contains a more complete and polished user and system interface.  It is
 not a complete superset; at this time it does not implement file archiving (better
 left to your backup solution) nor does it implement interface selection (better done
 with your own firewall rules and placement of the *-HOOK chains.)

Issues:
 Consider carefully whether you want to use this software and the full consequences
 to your site and\/or business.  By necessity, it will block potential customers and
 'good' connections along with villains.  You must consider the costs and benefits
 to your operation - the author does not endorse any specific policy.  In particular,
 the defaults should be viewed as examples, not value judgements.

 Very large numbers of exception IP blocks might benefit from implementing a subchain
 structure - but that would be a rather different use model.  The known use cases
 would probably be penalized - so one would want to make a dynamic choice.  I\'d
 want to see actual data before implementing this.

 The iptables-restore format is undocumented, though used by others.  It may be
 fragile.

 This code should use IPTables::IPv4 - but it doesn\'t currently work on my x64 system.
 It may be re-written to do so at some point.

 --tlhackque 1-Aug-2010, 8-Nov-2010, 3-Oct-2012
````
