# windapsearch
`windapsearch` is a Python script to help enumerate users, groups and computers from a Windows domain through LDAP queries. 
By default, Windows Domain Controllers support basic LDAP operations through port 389/tcp. With any valid domain account (regardless of privileges), it is possible to perform LDAP queries against a domain controller for any AD related information.

You can always use a tool like `ldapsearch` to perform custom LDAP queries against a Domain Controller. I found myself running different LDAP commands over and over again, and it was difficult to memorize all the custom LDAP queries. So this tool was born to help automate some of the most useful LDAP queries a pentester would want to perform in an AD environment.

### Requirements
`windapsearch` requires the `python-ldap` module. You should be able to get up and running qith two commands:

```
$ git clone https://github.com/ropnop/windapsearch.git
$ pip install python-ldap #or apt-get install python-ldap
$ ./windapsearch.py
```

## Usage
```
$ ./windapsearch.py -h
usage: windapsearch.py [-h] [-d DOMAIN] [--dc-ip DC_IP] [-u USER]
                       [-p PASSWORD] [-G] [-U] [-C] [-m GROUP_NAME] [--da]
                       [-s SEARCH_TERM] [-l DN] [-r] [--attrs ATTRS] [--full]

Script to perform Windows domain enumeration through LDAP queries to a Domain
Controller

optional arguments:
  -h, --help            show this help message and exit

Domain Options:
  -d DOMAIN, --domain DOMAIN
                        The FQDN of the domain (e.g. 'lab.example.com'). Only
                        needed if DC-IP not provided
  --dc-ip DC_IP         The IP address of a domain controller

Bind Options:
  Specify bind account. If not specified, anonymous bind will be attempted

  -u USER, --user USER  The full username with domain to bind with (e.g.
                        'ropnop@lab.example.com' or 'LAB\ropnop'
  -p PASSWORD, --password PASSWORD
                        Password to use. If not specified, will be prompted
                        for

Enumeration Options:
  Data to enumerate from LDAP

  -G, --groups          Enumerate all AD Groups
  -U, --users           Enumerate all AD Users
  -C, --computers       Enumerate all AD Computers
  -m GROUP_NAME, --members GROUP_NAME
                        Enumerate all members of a group
  --da                  Shortcut for enumerate all members of group 'Domain
                        Admins'. Performs recursive lookups for nested
                        members.
  -s SEARCH_TERM, --search SEARCH_TERM
                        Fuzzy search for all matching LDAP entries
  -l DN, --lookup DN    Search through LDAP and lookup entry. Works with fuzzy
                        search. Defaults to printing all attributes, but
                        honors '--attrs'

Output Options:
  Display and output options for results

  -r, --resolve         Resolve IP addresses for enumerated computer names.
                        Will make DNS queries against system NS
  --attrs ATTRS         Comma separated custom atrribute names to search for
                        (e.g. 'badPwdCount,lastLogon')
  --full                Dump all atrributes from LDAP.
```
### Specifying Domain and Account
To begin you need to specify a Domain Controller to connect to with `--dc-ip`, or a domain with `-d`.
If no Domain Controller IP address is specified, the script will attempt to do a DNS `host` lookup on the domain and take the top result.

A valid domain username and password are required for most lookups. If none are specififed the script will attempt an anonymous bind and enumerate the default namingContext, but most additional queries will fail.
The username needs to include the full domain, e.g. `ropnop@lap.example.com` or `EXAMPLE\ropnop`

The password can be specified on the command line with `-p` or if left out it will be prompted for.

### Enumerate Users
The `-U` option performs an LDAP search for all entries where `objectCategory=user`. By default, it will only display the commonName and the userPrincipalName.
The `--attrs` option can be used to specify custom or additional attributes to display, or the `--full` option will display everythin for all users.
WARNING: in a large domain this can get very big, very fast

Example:
```
$ ./windapsearch.py --dc-ip 10.9.122.100 -u jarrieta@cscou.lab -p nastyCutt3r -U
[+] Using Domain Controller at: 10.9.122.100
[+] Getting defaultNamingContext from Root DSE
[+]	  Found: DC=cscou,DC=lab
[+] Attempting bind
[+]	  ...success! Binded as:
[+]	  u:CSCOU\jarrieta

[+] Enumerating all AD users
[+]	Found 14 users:

cn: Administrator

cn: Guest

cn: krbtgt

cn: Joe Maddon
userPrincipalName: jmaddon@cscou.lab

<snipped...>
```

### Enumerate Groups and Group Memberships
Use the `-G` option to enumerate all entries where `objectCategory=group`. This will output the DN and CN of all groups.

To query group membership, use the `-m` option with either the DN or CN of the group you wish to query. The tool supports fuzzy search matching so even a partial CN will work. If it matches more than one group, the tool will specify which group to query.

Example:
```
$ ./windapsearch.py --dc-ip 10.9.122.100 -u jarrieta@cscou.lab -p nastyCutt3r -m empl
[+] Using Domain Controller at: 10.9.122.100
[+] Getting defaultNamingContext from Root DSE
[+]	Found: DC=cscou,DC=lab
[+] Attempting bind
[+]	...success! Binded as:
[+]	 u:CSCOU\jarrieta
[+] Attempting to enumerate full DN for group: empl
[+]	 Using DN: CN=employees,OU=cscou_groups,DC=cscou,DC=lab

[+]	 Found 10 members:

CN=Jason Heyward,OU=hitters,OU=cscou_users,DC=cscou,DC=lab
CN=Anthony Rizzo,OU=hitters,OU=cscou_users,DC=cscou,DC=lab
CN=Kris Bryant,OU=hitters,OU=cscou_users,DC=cscou,DC=lab
CN=Kyle Schwarber,OU=hitters,OU=cscou_users,DC=cscou,DC=lab
CN=Hector Rondon,OU=pitchers,OU=cscou_users,DC=cscou,DC=lab
CN=John Lackey,OU=pitchers,OU=cscou_users,DC=cscou,DC=lab
CN=Jon Lester,OU=pitchers,OU=cscou_users,DC=cscou,DC=lab
CN=Jake Arrieta,OU=pitchers,OU=cscou_users,DC=cscou,DC=lab
CN=Jed Hoyer,OU=managers,OU=cscou_users,DC=cscou,DC=lab
CN=Joe Maddon,OU=managers,OU=cscou_users,DC=cscou,DC=lab

[*] Bye!
```

#### Domain Admins
You can enumerate Domain Admins through two methods. One is to use `-m` with "Domain Admins". This will query LDAP for the "Domain Admins" entry and display all the members.

The more thorough way is to do a lookup of all users and determine if they or a group they belong to are part of "Domain Admins". This has the added benefit of discovering users who have inherited DA rights through nested group memberships. It's much slower, however.
See this link for more details on the technique used: https://labs.mwrinfosecurity.com/blog/active-directory-users-in-nested-groups-reconnaissance/
To do a recursive lookup for Domain Admins, you can use the "--da" option.

Example:
```
$ ./windapsearch.py --dc-ip 10.9.122.100 -u jarrieta@cscou.lab -p nastyCutt3r --da
[+] Using Domain Controller at: 10.9.122.100
[+] Getting defaultNamingContext from Root DSE
[+]	Found: DC=cscou,DC=lab
[+] Attempting bind
[+]	...success! Binded as:
[+]	 u:CSCOU\jarrieta
[+] Attempting to enumerate all Domain Admins
[+] Using DN: CN=Domain Admins,CN=Users.CN=Domain Admins,CN=Users,DC=cscou,DC=lab
[+]	Found 3 Domain Admins:

cn: Administrator

cn: Joe Maddon
userPrincipalName: jmaddon@cscou.lab

cn: Jed Hoyer
userPrincipalName: jhoyer@cscou.lab


[*] Bye!
```

### Enumerating Computers
LDAP queries can be used to enumerate domain joined computers. This is very useful when trying to build a list of targets without running a portscan or ping sweep.

Use the `-C` option to list all matching entries where `objectClass=Computer`. By default, the attributes displayed are 'cn', 'dNSHostName', 'operatingSystem', 'operatingSystemVersion', and 'operatingSystemServicePack'

If you specify the `-r` or `--resolve` option, the tool will perform a DNS lookup on every enumerated dNSHostName found and output the computer information, including IP address in CSV format. This can then be fed to other tools like nmap or CrackMapExec.

Example:
```
$ ./windapsearch.py --dc-ip 10.9.122.100 -u jarrieta@cscou.lab -p nastyCutt3r -C -r
[+] Using Domain Controller at: 10.9.122.100
[+] Getting defaultNamingContext from Root DSE
[+]	Found: DC=cscou,DC=lab
[+] Attempting bind
[+]	...success! Binded as:
[+]	 u:CSCOU\jarrieta

[+] Enumerating all AD computers
[+]	Found 5 computers:

cn, IP, dNSHostName, operatingSystem, operatingSystemVersion, operatingSystemServicePack
DC1,10.9.122.100,DC1.cscou.lab,Windows Server 2012 R2 Standard,6.3 (9600),
ordws02,10.9.122.9,ordws02.cscou.lab,Windows 8.1 Enterprise,6.3 (9600),
ORDWS01,10.9.122.5,ordws01.cscou.lab,Windows 7 Enterprise,6.1 (7601),Service Pack 1
ORDWS04,10.9.122.10,ORDWS04.cscou.lab,Windows 7 Enterprise,6.1 (7601),Service Pack 1
ORDWS03,10.9.122.7,ordws03.cscou.lab,Windows 10 Pro,10.0 (10240),

[*] Bye!
```

### Custom Searching
The tool allows for custom, fuzzy matching. You can perform a search and see results (DNs) with the `-s` option:

```
$ ./windapsearch.py --dc-ip 10.9.122.100 -u jarrieta@cscou.lab -p nastyCutt3r -s rizzo
[+] Using Domain Controller at: 10.9.122.100
[+] Getting defaultNamingContext from Root DSE
[+]	Found: DC=cscou,DC=lab
[+] Attempting bind
[+]	...success! Binded as:
[+]	 u:CSCOU\jarrieta
[+] Doing fuzzy search for: "rizzo"
[+]	Found 1 results:

CN=Anthony Rizzo,OU=hitters,OU=cscou_users,DC=cscou,DC=lab

[*] Bye!
```

To query the DN and display the attributes, use the lookup option, `-l`. You can provide this with a full DN, or a search term. If the search matches more than one DN, the tool will prompt you for which to use:

```
$ ./windapsearch.py --dc-ip 10.9.122.100 -u jarrieta@cscou.lab -p nastyCutt3r -l pitchers --attrs displayName
[+] Using Domain Controller at: 10.9.122.100
[+] Getting defaultNamingContext from Root DSE
[+]	Found: DC=cscou,DC=lab
[+] Attempting bind
[+]	...success! Binded as:
[+]	 u:CSCOU\jarrieta
[+] Searching for matching DNs for term: "pitchers"
[+]	 Using DN: OU=pitchers,OU=cscou_users,DC=cscou,DC=lab

displayName: Jake Arrieta

displayName: Jon Lester

displayName: John Lackey

displayName: Hector Rondon


[*] Bye!
```

### Credits
Heavily influenced by this post and research done by MWR Labs:
https://labs.mwrinfosecurity.com/blog/active-directory-users-in-nested-groups-reconnaissance/

and their tool to perform offline querying of LDAP:
https://labs.mwrinfosecurity.com/blog/offline-querying-of-active-directory/




