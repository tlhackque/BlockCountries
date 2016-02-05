BlockCountries
================

iptables manager to block by country

Copyright (C) 2010, 2012, 2013, 2015, 2016 Timothe Litt

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

#Latest updates:

V2.10 Add Default-Start and Default-Stop to LSB block

V2.9 Simplify configuration so it's not necessary to modify the script.
     Improve documentation and packaging.  `bcinstall` checks for minimum
     versions and iptables configuration.

V2.8 Add $LOGLEVEL to control syslog priority

V2.7 Add LSB init block

V2.6 Add version command. Syntax cleanup. Corner case fix for help.

V2.5 Fix uninitialized variable when no command arguments specified.

V2.4 Documentation updates, no functional changes.
* Clarify -permitonly and rules placement/priorities

V2.3 Make conntrack optional - older netfilters are still around
* Use -conntrack if you see warnings (or errors)
* Add -passive to use passive FTP connection when getting registry data.  This usually traverses firewalls more easily.

V2.2 Support conntrack instead of state

V2.1 Support new statistics file used by registries

#Dependencies:
The script uses a number of Perl library modules, available
from CPAN (or in the base Perl distribution).  The
`bcinstall` bash script should be used to determine whether
they (and Perl) are installed. 

`bcinstall` will also verify that the minimum required version of each
module is installed.  Later versions should be used if available.

There are known bugs that impact BlockCountries in earlier versions
of several modules.  Don't try to use any version less than the
minimum that bcinstall checks for.

#Cautions:

Consider carefully whether you want to use this software and the full consequences
to your site and/or business.  By necessity, it will block potential customers and
'good' connections along with villains.  You must consider the costs and benefits
to your operation - the author does not endorse any specific policy.  In particular,
the defaults should be viewed as examples, not value judgements.

The IPv6 optimizations have not been tuned; it is not clear what the right
subchain structure is.

#Installation:

Unpack the `tar.gz` or `zip` file that you downloaded from github.
(`tar.gz` files are available from https://github.com/tlhackque/BlockCountries/releases)

This will create a subdirectory named BlockCountries-<version>.

cd to that directory.

Run `bcinstall` (see its README) to check that Perl and its
dependencies are installed and meet the minimum version requirements.

Copy BlockCountries to `/etc/init.d` (or your distributions startup directory)
run `chkconfig`, `systemctl enable`, `update-rc.d` (or equivalent) to
include it in the automatic system startup.

If Perl is not installed in `/usr/bin`, create a softlink to it there,
`bcinstall` will provide guidance.

On Unix-like systems,

which perl

will usually reveal the location.

The file `config/BlockCountries` should be copied to your distribution's
configuration directory, such as `/etc/sysconfig` or `/etc/default`.

BlockCountries looks for the configuration file in several common directories,
Use BlockCountries help for a list. 

If you must place the configuration file in an alternate location, define
the environment variable `BlockCountriesCFG` in all the environments where
BlockCountries will be run.  (cron, startup, and interactive)
This is the first file attempted.

Inspect the first few lines of the configuration file and modify the variables
in Section 1 for your environment.

Verify that BlockCountries finds your file by using `BlockCountries config`

Add a crontab entry similar to:
````
CRONJOB=1
11 23 * * Sun /etc/init.d/BlockCountries start -update
````
Please pick a different time.  Note that IP address allocations are fairly
stable; updating more than once a week is rarely productive.


Run
````
BlockCountries -start -update
````

The filter should download the latest IP country assignments and be in place.


#Usage:

The usage as of V2.4 follows (the words prefixed by $ are filled
in at runtime).  Later versions differ - the definitive version,
which is customized for your system, is available from:
````
$prog help
````

The following usage provides an introduction to BlockCountries.

Please do not use this version of the usage to configure BlockCountries
for production; use the version from the `help` command, which is
always up-to-date with the latest functions and clarifications.

````
IP filter manager for country filters

Usage: $prog command args
  config             Display active configuration file
  status [-v]        Display filter status.
                     -v provides configuration from config file
                     and command file - NOT iptables.
  list               List available country names/codes.
                     Contacts server for list.
  intercepts [-host name] [-days n]
                     List today's intercepts by host from $LOG.
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
 -conntrack         Use -m conntrack rather than -m state to avoid warnings from newer iptables
 -log               Install a logging rule to log rejected packets.
 -nolog             Don't install a logging rule. (default)
 -nolimit           Do not limit logging (can generate huge log files if under attack; not advised)
 -limit spec        Limit logging, default = $loglimits[0]:$loglimits[1] (see man iptables "limit")
 -atport n          Allow connections to TCP port n even FROM banned addresses.
                    May specify any number of times.  May use a service name.
 -auport n          Allow connections to UDP port n even FROM banned addresses.
                    May specify any number of times.  May use a service name.
 -atporto n         Same as -atport, but for connections TO banned addresses.
 -auporto n         Same as -auport, but for connections TO banned addresses.
 -aip ip(/mask)    Allow connection from an otherwise banned IP address.
                    For a block, specify a netlength or mask. A hostname may
                    also be specified.  May specify any number of times.
 -dip ip(/mask)    Deny connections from an otherwise allowed IP address.
                    Same syntax as -aip.
 -passive           When using FTP for updates, use passive mode (traverse firewalls)
 -permitonly        Listed countries will be permited, all others denied.
                    Put in -permitonly in the configuration file so status and start are consistent.
 -blockout          Also generate rules to block output & forwarded-output.
                    This is probably not required for most applications, and
                    will roughly double the memory requirements.
                    Caution: If you use -blockout for start, you must also use it for stop.
                    This will not be a problem if it's in the configuration file.
 -d                 Output random debugging messages
 -v                 Output extended status/statistics
  CC                ISO Country code or name to ban (or permit if -permitonly).
                    Specify as many as you like.
                    Default list:

Arguments may also be obtained from the configuration file,
The configuration file must be readable but not
executable.  It is searched for in a number of places. See
BlockCountries help for a list.  The first file found is used:

The system configuration is specified by the variables:
 -logfile    "/var/log/messages*"  # Wildcard should include rotated/archived/.gz compressed files
                                   # containing iptables log messages
 -loglevel number or name          # syslog message priority.  Normally omitted. System-dependent.
                                   # Usually 0-6 (or as names, EMERGENCY, ALERT, CRITICAL,ERROR,
                                   # WARNING,NOTICE,INFORMATIONAL, or DEBUG)
 -logpfx '[Blocked CC]:'           # Prefix used for log messages from BlockedCountries
 -logpgm 'kernel'                  # Program name used by netfilter.  Always 'kernel'.
 -path   '/sbin'                   # Path for iptables components
 -zonedir '/root/blockips'         # Directory used to hold IP mapping data

The defaults are indicated above.  These variables can NOT be specified on the command line.

Anything else (except comments) contained in it is prepended to every command
line's arguments.

Use single or double-quoted strings for country names containining spaces.

This script is designed to run as a service; depending on your distribution
`chkconfig`, `update-rc.d`, or `system-ctl enable` will link it into
`/etc/rcN.d/.`  Be sure to put all configuration in the configuration file,
since startup scripts only get "start" (or "stop") as an argument.

This script should also be run with start -update from a cron job - weekly
is suggested - to obtain the latest IP address databases.  If the CRONJOB
environment variable is set, only errors and zone updates will be reported.
This is recomended to minimize e-mail from cron.

To prevent communications from banned IP addresses during updates,
start will install new rules before removing active rules.  For
this to be effective, you should not stop the service.

The status -v command will report the current configuration, although the
actual implementation in iptables will be different due to optimizations.

Rules are applied to the INPUT, OUTPUT or FORWARD chains after any existing
IPTABLES rules.  Typically, this means that related and established connections
are accepted before BlockCountries rules are applied, and icmp packets are
allowed.  (This varies by distribution and local option.)  To control
placement, define the corresponding -HOOK chain.(e.g. INPUT-HOOK) and call it
it from the main chain(s).  BlockCountries rules will be installed in
the -HOOK chain instead of the main chain.

When manual configuration directives are used, the rules are applied in the
following order (first match determines result):
   -atports (or -atportso) are accepted
   -auports (or -auportso) are accepted
   -aips are accepted
   -dips are denied
   Country constraints deny or accept (per -permitonly)
   IPTABLES rules later in the caller's chain
If a BlockCountries rule denies a connection, it will be dropped (and logged).
If a BlockCountries rule "accepts" a connection, it is still subject to any
IPTABLES rules that follow it.

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
 rules for the default banned address list.  It's also somewhat faster than a shell
 script, and contains a more complete and polished user and system interface.  It is
 not a complete superset; at this time it does not implement file archiving (better
 left to your backup solution) nor does it implement interface selection (better done
 with your own firewall rules and placement of the *-HOOK chains.)

Issues:
 Consider carefully whether you want to use this software and the full consequences
 to your site and/or business.  By necessity, it will block potential customers and
 'good' connections along with villains.  You must consider the costs and benefits
 to your operation - the author does not endorse any specific policy.  In particular,
 the defaults should be viewed as examples, not value judgements.

 Very large numbers of exception IP blocks might benefit from implementing a subchain
 structure - but that would be a rather different use model.  The known use cases
 would probably be penalized - so one would want to make a dynamic choice.  I'd
 want to see actual data before implementing this.

 The iptables-restore format is undocumented, though used by others.  It may be
 fragile.

 This code should use IPTables::IPv4 - but it doesn't currently work on my x64 system.
 It may be re-written to do so at some point.

 --tlhackque 1-Aug-2010, 8-Nov-2010, 3-Oct-2012, 4-Sep-2013, 17-Dec-2015, 5-Feb-2016
````

#Bug reports and suggestions

Please raise bug reports or suggestions at http://github.com/tlhackque/BlockCountries/issues.

Always include `BlockCountries version` `BlockCOuntries config`, and `perl --version`.

Suggestions and/or praise are also welcome.
