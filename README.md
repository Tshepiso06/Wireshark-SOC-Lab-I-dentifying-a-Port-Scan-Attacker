# Wireshark-SOC-Lab-I-dentifying-a-Port-Scan-Attacker

A hands-on SOC analyst exercise: capture live traffic, find the attacker, and identify the attack type using Wireshark.

Lab source: Networkwalks – Wireshark Lab 3: Identify the Attacker's IP. This repo is my own write-up of the walkthrough and what I learned. Credit for the lab environment goes to Networkwalks.

Scenario

A staff member reported unusual traffic to the company web server. As the SOC analyst, I had to capture the packets, work out what type of attack was happening, and identify the attacker's IP address.

Topology

Role	Device	IP
SOC Analyst	Windows 11 PC running Wireshark	192.168.10.10
Web Server (target)	Company web server	192.168.10.20
Attacker	Unknown at the start	?
Capture point	Switch with SPAN (port mirror)	n/a

Tools

	•	Wireshark
	•	Display filters
	•	Statistics → Conversations

Result Summary

Finding	Value
Attacker	192.168.10.66
Target	192.168.10.20 (web server)
Attack type	Port scan (TCP SYN scan)
Ports probed	40
Open ports	22, 80, 443
Closed ports	37 (server replied RST, ACK)
Attacker source port	52311 (same in every probe)
Total packets captured	309
SYN packets	73

Step-by-Step Walkthrough

Step 1: Capture the traffic

	1.	Sign in to the Windows PC.
	2.	Open Wireshark and double-click the WiFi interface (the one showing an activity graph).
	3.	Let the capture run, then read the packet count in the status bar.

Result: 309 packets captured.

Step 2: Isolate connection attempts (SYN packets)

Most traffic is normal. A TCP connection starts with a SYN packet, so filter for those:

tcp.flags.syn == 1

Result: 73 packets displayed.

Step 3: Find the attacker

Scroll the Source column of the filtered results. Normal hosts send only one or two SYN packets. One IP sends a long block of them in a fraction of a second, starting at about 3.4 seconds.

Result: the attacker is 192.168.10.66.

Step 4: Identify the target

Look at the Destination column for that block of packets.

Result: the target is 192.168.10.20, the web server.

Step 5: Check the source port

In the Info column, the number before the arrow is the source port.

Result: the source port is 52311 in every packet, while the destination port changes each time. This is typical of scanning tools: one fixed source port, many different destination ports.

Step 6: Count the ports probed

	1.	Go to Statistics → Conversations.
	2.	Open the TCP tab.
	3.	Find the rows between 192.168.10.66 and 192.168.10.20. Each row is one port.

Result: 40 ports probed.

Step 7: Find the open ports

Filter all traffic involving the attacker:

ip.addr == 192.168.10.66

An open port answers a SYN with a SYN, ACK. Look for [SYN, ACK] packets coming from the server. The number before the arrow is the open port.

Result: ports 22 (SSH), 80 (HTTP), 443 (HTTPS) are open.

Step 8: Find the closed port reply

For the other ports, the server replies with a different flag combination.

Result: [RST, ACK]. The server uses a reset to say the port is closed.

Step 9: Count the closed ports

40 ports probed − 3 open = 37 closed ports.

Step 10: Identify the attack type

One IP tried many different ports in a very short time and sent no actual data. This is a port scan, not a SYN flood. A SYN flood hammers one port with thousands of packets; a port scan spreads probes across many ports to map what is open.

Attack Timeline

Attacker (192.168.10.66:52311)  --SYN-->       Server:22     --> SYN, ACK  (open)
Attacker (192.168.10.66:52311)  --SYN-->       Server:80     --> SYN, ACK  (open)
Attacker (192.168.10.66:52311)  --SYN-->       Server:443    --> SYN, ACK  (open)
Attacker (192.168.10.66:52311)  --SYN-->       Server:other  --> RST, ACK  (closed)

Useful Filters

Purpose	Filter
All SYN packets	tcp.flags.syn == 1
SYN only (no ACK), i.e. new connection attempts	tcp.flags.syn == 1 && tcp.flags.ack == 0
Everything from/to the attacker	ip.addr == 192.168.10.66
Open port replies	tcp.flags.syn == 1 && tcp.flags.ack == 1
Closed port replies	tcp.flags.reset == 1

SOC Analyst Takeaways

	•	Indicators of a port scan: one source, many destination ports, very short time window, no data payload.
	•	Open vs closed: SYN, ACK means open. RST, ACK means closed.
	•	Fixed source port + changing destination port is a strong scanning signature.
	•	MITRE ATT&CK mapping: Reconnaissance / Discovery, T1046 Network Service Discovery and T1595.001 Active Scanning: Scanning IP Blocks.

Recommended Response (what a SOC would do next)

	1.	Document the source IP, target, ports and timestamps.
	2.	Check whether 192.168.10.66 is an authorised scanner (e.g. a vulnerability management tool). If not, treat it as a possible compromised or rogue internal host.
	3.	Isolate or block the source if unauthorised.
	4.	Review exposed services on the target. Confirm 22, 80 and 443 are all meant to be reachable, and restrict SSH (22) to trusted IPs.
	5.	Add detection: alert on one source hitting many distinct destination ports within a short window.

What I Learned

	•	Using display filters to cut a capture down to the traffic that matters.
	•	Reading TCP flags (SYN, SYN-ACK, RST-ACK) to tell open ports from closed ones.
	•	Using Statistics → Conversations to quantify an attack.
	•	Telling apart a port scan and a SYN flood.

Repo Structure

.
├── README.md
└── screenshots/
    ├── 01-capture.png
    ├── 02-syn-filter.png
    ├── 03-attacker.png
    └── 04-conversations.png

Author

Tshepiso Mofokeng
Aspiring SOC Analyst | BCom Information Technology and Management (MANCOSA) | Johannesburg, South Africa

LinkedIn [https://www.linkedin.com/in/tshepiso-mofokeng-34782a281/]
