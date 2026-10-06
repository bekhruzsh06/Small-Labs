DNS Lab



## **Context**

**Domain** - entire namespace subtree. For example, if you own `microsoft.com`, the domain includes root `microsoft.com` and every subdomain beneath it


**Zone** - A specific portion of a domain that is managed together on a single name server

**Master file** - File that gets copied by secondary file. Original file that contains zone configurations

**Secondary file** - File that serves as a backup and copies content from master file

## **Structure**

Ubuntu - Authoritative Name Server
Kali Linux - attacking machine

## **1. Environment Set Up**

Firstly we need to isolate our environment in a single subnet on VMWare

1. Edit -> Virtual Network Adapter
2. Select unused network or creare one (VmNet2)
3. Set it to Host-Only 
	 - **Bridged/NAT:** Connects your VM to the internet through your home Wi-Fi.
    
	- **Host-Only:** Creates a virtual network switch (a virtual hub) inside your laptop. The VMs plugged into this switch can talk to each other, but **they cannot reach the internet**, and devices on your home Wi-Fi cannot reach them.

**Download Ubuntu**

- **RAM:** 2 GB (DNS is very lightweight).
    
- **CPU:** 2 Cores.
    
- **Network Adapter:** Set to Custom -> `VMnet2` (the one you just created). Install Ubuntu with default settings. Once installed, assign it a static IP (e.g., `192.168.50.10`)


## Phase 2: Building the DNS Server (BIND9)

Now we will turn your Ubuntu VM into an authoritative DNS server for a fake domain, let's call it `bek.local`. Run these commands inside your Ubuntu VM:

**1.Install BIND9:**

Update your packages and install the BIND9 package and DNS utilities.


```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc dnsutils
```

**Ubuntu Server**

- `bind9` **(Berkeley Internet Name Domain v9):** The core DNS server software. Responsible for listening on port 53 and responding to queries with IP mappings

- `bind9utils`: A collection of administrative tools for managing and testing BIND9, including tools like `named-checkconf` (which verifies configuration syntax before restarting the server) and `rndc` (remote name daemon control)
- `dnsutils`: A package containg fundamental lookup tools: `dig`, `nslookup`
- `net-tools`: A legacy networking toolkit, providing commands such as `ifconfig`, `netstat`, `route` and `arp`


**Kali tools**

- `dnsenum`: Automated script to enumerate DNS. Resolves domain names, queries MX/NS records, attempts AXFR zone transfers, runs dictionary-based brute-force attacks

- `fierce`: A lightweight Python-based reconnaissance tool that locates non-contigous IP spaces and subdomains across a target domain using wordlists and DNS queries.

- `dsniff`: Suite of low-level network pentesting tools. Includes utilities like `arpspoof` (to poison the ARP cache and place your machine in the middle of network traffic) and `dnsspoof` (to forge DNS reply packets and redirect victims to malicious IPs).

- **`ettercap-text-only`:** A command-line version of Ettercap, a comprehensive Man-in-the-Middle (MitM) framework. It combines live connection sniffing, ARP poisoning, traffic filtering, and protocol-specific attacks like DNS spoofing into a single tool without requiring a GUI.

## **2. Configuring DNS conf Files**


### 2.1. Global Options File: `/etc/bind/named.conf.options`

**Definition and Purpose:**

Main configuration file that dictates global behaviour of BIND9 DNS server

```text
options {
        directory "/var/cache/bind";
        recursion no;
        listen-on { any; };
        allow-query { any; };
};

```

- **`options {`**: Opens the global options block. Everything inside these braces applies to the server as a whole.
    
- **`directory "/var/cache/bind";`**: Sets the working directory. If BIND9 needs to write temporary

- cache files or dump databases, it will save them in this folder.
    
- **`recursion no;`** Disables recursive DNS resolution. Tells server not to fetch external domains (google.com). It'll answer only for queries for domains it explicitly knows.

- `listen-on { any; };` Listen on all interfaces on port 53

- `allow-query { any; };` ACL that allows any address to ask this server for DNS records

### 2. Local Zone Declaration: `/etc/bind/named.conf.local`

**Definition and Purpose:**

Used to declare the specific domains (known as "zones") that DNS server manages. Acts as an index, telling the BIND9 service which domains it is authorative for and exactly where on the hard drive it can find the database files containing the IP addresses for those domains.

**File Content:**

```Plaintext
zone bek.local {
	type master;
	file "/etc/bind/zones/db.bek.local"
	allow-transfer { any; };
};
```

**Line-by-Line Explanation:**

- **`zone bek.local`** - declares new DNS zone for the domain bek.local and opens its config block

- **`file "/etc/bind/zones/db.bek.local";`**: Provides the absolute file path pointing to the text file that contains the actual DNS records (A, TXT, MX) for this zone.
    
- **`allow-transfer { any; };`**: Dictates which IP addresses are allowed to perform a full zone transfer (AXFR), which copies the entire database. _Note: Setting this to `any;` is a deliberate security vulnerability added for your lab so you can practice zone transfer attacks from Kali._

## 3. Zone Data File: `/etc/bind/zones/db.bek.local`

**Definition and Purpose:**

This is the actual "forward lookup" database for your domain. Raw text file containing the Resources Records, that maps hostnames (like `www` or `portal`) to their specific IP addresses along with metadata about the domain itself



```Plaintext
$TTL    604800
@       IN      SOA     ns1.bek.local. admin.bek.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.bek.local.
ns1     IN      A       192.168.50.129
www     IN      A       192.168.50.15
portal  IN      A       192.168.50.16
secret  IN      A       192.168.50.99
flag    IN      TXT     "FLAG{dns_zone_transfer_success}"
```

**Line-by-Line Explanation:**

**`$TTL 604800`**: Defines the default Time-To-Live for all records in this file. 604800 seconds equals one week. This tells clients how long they can cache a record before they must query your server again.

-  `@       IN      SOA     ns1.bek.local. admin.bek.local.` Starts the SOA record 
	- `@` Represents the root of the domain (`bek.local`)
	- `IN` stands for Internet protocol class
	- `SOA` defines the primary nameserver (`ns1.bek.local`). and the administrator's email address (the first dot acts as the `@` symbol, meaning `admin@bek.local`). The `(` opens a block for timing parameters.

- **`2 ; Serial`**: The version number of this file. If you add a new record, you must increase this number so secondary servers know an update occurred.
    
- **`604800 ; Refresh`**: Tells secondary (slave) servers to check this primary server for updates every 7 days.
    
- **`86400 ; Retry`**: If a secondary server tries to check for updates and fails, it should wait 1 day (86400 seconds) before trying again.
    
- **`2419200 ; Expire`**: If a secondary server cannot reach this primary server for 28 days, it will stop answering queries for this domain.
    
- **`604800 ) ; Negative Cache TTL`**: If a client asks for a subdomain that does _not_ exist, this dictates how long they should cache the "not found" response. The `)` closes the SOA block.
    
- **`;`**: A semicolon acts as a comment line in BIND9.

- **`@ IN NS ns1.hackmelab.local.`**: The Name Server (NS) record. It explicitly states that `ns1.hackmelab.local` is the official nameserver for this domain.

- `ns1 IN A 192.168.50.10`: An Address (A) record mapping the hostname `ns1` directly to your Ubuntu server's IP address.
    
- **`www IN A 192.168.50.15`**: An Address (A) record mapping `www` to a fake web server IP in your lab.
    
- **`portal IN A 192.168.50.16`**: An Address (A) record mapping `portal` to another IP.
    
- **`secret IN A 192.168.50.99`**: A "hidden" subdomain that wouldn't normally be found without brute-forcing or executing a zone transfer.
    
- **`flag IN TXT "FLAG{dns_zone_transfer_success}"`**: A Text (TXT) record. In the real world, this is used for domain ownership verification (like SPF). In this lab, it acts as a Capture The Flag (CTF) prize for successfully executing a zone transfer.



## **3. File Structure and Connection**


```Plaintext
[ named.conf.options ]  ──> Sets the global rules (port 53, listen on all interfaces, no recursion).
         │
         ▼
[ named.conf.local ]    ──> The "Index Book": tells BIND, "If someone asks for bek.local, 
         │                  load the data from /etc/bind/zones/db.bek.local."
         ▼
[ db.bek.local ]        ──> The "Phonebook": contains the actual names and IP addresses 
                            (ns1, www, portal, secret, flag).
```

- **File 1 (`named.conf.options`): The Engine Rules**
    
    - It does not know or care about `bek.local`. It just sets baseline rules like: _"Listen on all network adapters"_ and _"Allow anyone to ask questions."_
        
- **File 2 (`named.conf.local`): The Domain Index / Router**
    
    - This is the link between the BIND service and your database file. It says: _"We are the master authority for `bek.local`. Go read the file `/etc/bind/zones/db.bek.local` to get the records."_
        
- **File 3 (`db.bek.local`): The Raw Database (The Zone File)**
    
    - This file contains the actual list of hosts and IPs.
