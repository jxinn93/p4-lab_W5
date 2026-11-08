# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **NNNN**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

     Loop was created by changing the dst_id = 3 entry in the MyIngress.myTunnel_exact table on s2. The output port was change to port 1, which sends the packet back to s1. Since s1 already forwards tunnel exit 3 to s2, the packet continuously travels between the two switches.

     Command used: 
     sh ./p4cli s2 table_dump MyIngress.myTunnel_exact
     sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"
     h1 python3 send.py 10.64.0.3 "loop" --tunnel 3

     What did the switch logs show over time, with numbers from loop-before.txt?
     In loop-before.txt, the number of packets processed by s1 and s2 keep increasing. This shows that the packet keep looping between the switches without being stopped.

     Then look at the IPv4 path in tunnel.p4: it decrements the TTL. Does that stop an IPv4
     No, because even ipv4_forward decrease the IPv4 TTL by 1, but it does not check whether the TTL has reached zero or not. So, it does not properly stop an IPv4 forwarding loop in this code.

## 2. Design: the hop limit

     An 8-bit hop_limit field added after dst_id in the custom tunnel header. Reason choosing 8 bit becuase it support values from 0 to 255 which is sufficient for the number of switches where a packet can pass through.

     8 is selected as the default starting value because it allows a packet to go through several switches while still limiting how far it can go. This value can be change using the --hops argument.

     The hop_limit is decrease in the ingress pipeline, every switch will check the hop_limit before sending the tunnel packet out. If the value is zero, the packet will be dropped.

     Sender.py and receive.py need to be updated to understand the new header format to prevent forwarding error.The parser extracts the additional field, while the deparser emits the complete tunnel header.  

## 3. Implementation

     tunnel.p4: Added the 8-bit hop_limit field to the tunnel header and updated the ingress pipeline to decrease the value at each switch and drop packets when it reaches zero.

     send.py: Added the --hops argument so the sender can set the starting hop limit. The default value is 8.

     receive.py: Updated the Scapy tunnel-header definition to include hop_limit and display the remaining value when a packet arrives.

## 4. Evidence: it works

-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     Listening on h2-eth0...
     Received (tunnelled, dst_id=2,hop_limit=0) from 10.64.0.1 to 10.64.0.2, TTL 64: 'test 1'
     Received (tunnelled, dst_id=2,hop_limit=3) from 10.64.0.1 to 10.64.0.2, TTL 64: 'test 1'
     Received (tunnelled, dst_id=2,hop_limit=3) from 10.64.0.1 to 10.64.0.2, TTL 64: 'to h2'
     Listening on h3-eth0...
     Received (tunnelled, dst_id=3,hop_limit=3) from 10.64.0.1 to 10.64.0.3, TTL 64: 'to h3'

     - plain IPv4 still works (delivery.txt);
     PING 10.64.0.3 (10.64.0.3) 56(84) bytes of data.
     64 bytes from 10.64.0.3: icmp_seq=1 ttl=62 time=5.82 ms
     64 bytes from 10.64.0.3: icmp_seq=2 ttl=62 time=12.5 ms

     --- 10.64.0.3 ping statistics ---
     2 packets transmitted, 2 received, 0% packet loss, time 1001ms rtt min/avg/max/mdev = 5.819/9.149/12.479/3.330 ms

     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->
       s1:990, s2998, yes, match by design where the loop stop as expected when its hop limit reached 0.
       sh grep -c "Processing packet received" logs/s1.txt logs/s2.txt > assignment/evidence/loop-after.txt
       logs/s1.txt:990
       logs/s2.txt:988

       mininet> sh sleep 5; grep -c "Processing packet received" logs/s1.txt logs/s2.txt >> assignment/evidence/loop-after.txt
       logs/s1.txt:990
       logs/s2.txt:988

## 5. Evaluation

The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to
     
        I predicted that old s2 won't able to understand the new header because miss out hop_limit then pogram end with error. 

        However, the program can still be executed. Only h2 received no packet (0 packets captured) in h2cap.txt. The switch log counts also continued increasing, showing that the packet was still looping between s1 and s2. This happened because old s2 still processed the packet using its old header format but misunderstood the extra byte. Therefore, the loop continued because old s2 did not decrease the hop limit.

     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).

        Upgrade all switches together: Every switch is upgraded to the new program at the same time. This avoids compatibility problems, but requires maintenance planning and may cause downtime.

        Use a versioned header: Add a version field to identify different header formats. New switches can support both old and new formats during the migration. This allows gradual upgrades but requires more complicated parser logic.

     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?

        The control plane check_paths() checks the network paths to detect possible loops before packet are forwarded. However, it may not detect unexpected loops caused by network failures or incorrect forwarding rules.

        The data plane (hop_limit) prevents packets from travelling in circles forever. Every time a packet passes through a switch, its hop limit decrese by 1 and drop when limit hit zero.

        The main difference is that check_paths() detects potential loops in the network, while the hop_limit stops packets from looping during forwarding.

        Real networks use both because detecting a problem early is better but having extra protection is also necessary in case something unexpected happens.

     d) What would this change cost on a hardware switch running at line rate? -->
     Adding a hop-limit check means the switch needs to perform extra operations for every packet include reading hop_limit value, decrease value, check whether is has reach zero and drop if zero.

## AI use

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->

     Chatgpt used to generate the code modify based on the request I give (set the hop_limit and starting value to 8). 

     Chatgpt help me verify my work is correct by sending the output to it.

     Gemini help to explain the concept, and reorganize my answer into more structure sentence.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
