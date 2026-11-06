# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **4919**

## 1. Analysis: the loop

I created a forwarding loop by changing the tunnel forwarding entry for destination ID 3 on switch s2. Normally, s2 forwards this traffic to h3 through port 3. I changed the entry to forward it through port 1, which sends the packet back to s1.

The commands used were:

```text
sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"
h1 python3 send.py 10.19.0.3 "loop" --tunnel 3
```

The switch logs showed continuous packet processing. In one three-second interval, s1's logged count increased from 448 to 654, while s2's increased from 446 to 652. Both switches therefore recorded 206 additional packet-processing events. Some earlier log counts decreased, likely because the logs were truncated or rotated.

The original tunnel forwarding action did not decrease any hop counter, so the packet could continue circulating.

For ordinary IPv4 forwarding, `ipv4_forward` decrements the IPv4 TTL by one. However, the program does not explicitly check for TTL reaching zero before forwarding. Since TTL is an 8-bit field, decrementing zero can wrap it to 255. Therefore, this implementation does not guarantee that an IPv4 loop will stop.

Evidence: `assignment/evidence/loop-before.txt`.

## 2. Design: the hop limit

I added an 8-bit `hop_limit` field after `dst_id` in the custom tunnel header. The tunnel header now contains a 16-bit protocol ID, a 16-bit destination ID, and an 8-bit hop limit.

I placed the new field at the end so the existing `proto_id` and `dst_id` fields keep their original positions. This also allows the old switch to continue reading those fields during mixed deployment.

The sender uses a default initial hop limit of 4. This is enough for the normal path through s1 and s2, while limiting how long a packet can circulate if a forwarding loop occurs. The sender also allows the user to specify a different value using `--hops`.

In the P4 ingress pipeline, a switch checks the hop limit before applying the tunnel forwarding table. If the value is 1 or less, the packet is dropped. Otherwise, the switch subtracts one and continues forwarding.

The parser and deparser use the updated tunnel header definition. I also updated the sender and receiver to include and interpret the new field consistently.

## 3. Implementation

I modified three main files.

**`tunnel.p4`:** Added `bit<8> hop_limit` to the tunnel header. In the ingress pipeline, tunnel packets are dropped when the hop limit is 1 or less. Otherwise, the value is decremented before applying `myTunnel_exact`. Ordinary IPv4 traffic continues to use `ipv4_lpm`.

**`send.py`:** Added a `ByteField` for the hop limit in the Scapy tunnel definition. The `--hops` argument sets the initial value, with a default of 4 and an allowed range of 1–255. The existing `--tunnel` option remains available.

**`receive.py`:** Added the same hop-limit field to the receiver's Scapy definition. The receiver prints the remaining hop limit when it receives a tunnelled UDP packet.

I also preserved the original P4 implementation as `tunnel_v1.p4` for mixed-deployment testing.

## 4. Evidence: it works

I first tested normal tunnel delivery using an initial hop limit of 4.

| Test | Result |
|---|---|
| h1 to h2, tunnel destination 2 | Received with hop limit 2 |
| h1 to h3, tunnel destination 3 | Received with hop limit 2 |
| Plain IPv4 ping from h1 to h3 | 2 packets received, 0% packet loss |

The tunnel packets passed through two switches, so their hop limits decreased from 4 to 2. Their IPv4 TTL remained 64. Ordinary IPv4 ping also worked, with a received TTL of 62.

I then recreated the same forwarding loop used in Section 1. With an initial hop limit of 4, the expected switch sequence was s1, s2, s1, then s2 dropping the packet when its hop limit reached 1.

The recorded switch-log counts were 1,434 for s1 and 1,433 for s2. After five seconds, both counts remained unchanged. This is consistent with the forwarding loop terminating rather than continuing indefinitely. These counts include other logged traffic and do not independently prove the exact number of visits by the test packet.

Evidence: `assignment/evidence/delivery.txt` and `assignment/evidence/loop-after.txt`.

## 5. Evaluation

### a) Mixed deployment

I tested a mixed deployment where s1 used the updated `tunnel.p4` and s2 used the original `tunnel_v1.p4`.

I initially expected the additional hop-limit byte to cause a compatibility problem because the old switch did not define that field. However, the packet was delivered successfully.

The new tunnel header keeps the original four bytes in the same order: two bytes for `proto_id` and two bytes for `dst_id`. The fifth byte contains `hop_limit`. The old switch can still read the original fields and select the forwarding action. In this experiment, the additional byte remained in the packet, allowing the receiver to interpret the updated header.

The packet started with a hop limit of 4 and reached h3 with a hop limit of 3. This showed that s1 decremented the value, while s2 forwarded the packet without changing it.

Ordinary IPv4 forwarding also worked, with two successful pings and 0% packet loss.

I then recreated the forwarding loop. The recorded processing counts were 1,443 for s1 and 1,441 for s2, and neither increased after five seconds. This is consistent with the loop terminating because s1 still enforces the hop limit.

However, loop protection is weaker during mixed deployment. Only s1 decrements the hop limit, so the packet can make more switch visits before being dropped. A loop involving only old switches would not receive this protection.

Evidence: `assignment/evidence/mixed.txt`.

### b) Safe rollout strategy

One approach is to add a version field to the tunnel header. Switches could identify the packet format and use the appropriate parser. This provides explicit version identification but requires support for multiple header formats.

Another approach is to assign a new EtherType to the updated tunnel format. Old and new packets would then be distinguishable, allowing switches to handle or reject unsupported versions. However, this requires updating forwarding and parsing rules throughout the network.

A third approach is to append new fields after the existing fields, as done in this assignment. This preserves the positions of the original fields and may allow older switches to forward packets without modification. However, older switches cannot enforce the new functionality, and compatibility must be tested.

For a larger network, I would prefer explicit versioning or a new EtherType, together with staged upgrades and compatibility testing. I would verify the forwarding behavior of each stage before enabling the new format throughout the network.

### c) Data-plane versus control-plane loop protection

The hop limit provides protection in the data plane while packets are being forwarded. It can stop a circulating packet even if an incorrect forwarding entry creates a loop.

Week 3's `check_paths()` operates in the control plane by checking route information before applying it to routers. It can identify problems in the planned routing configuration, but it may not detect every unexpected forwarding problem that occurs later.

The two approaches complement each other. Control-plane validation helps prevent incorrect routes from being installed, while data-plane hop limits limit the damage if a loop still occurs.

### d) Hardware cost at line rate

The updated tunnel header adds one byte to each tunnelled packet. A hardware switch must also extract the field, compare it with the drop threshold, and decrement it when forwarding.

These are simple operations that can normally be implemented using parser and match-action pipeline resources. However, the extra header field and processing logic still consume hardware resources. Whether line-rate performance is maintained depends on the switch's hardware capabilities and pipeline constraints.

## AI use

I used ChatGPT to explain P4 forwarding behavior, help plan the hop-limit design, provide implementation guidance for `tunnel.p4`, `send.py`, and `receive.py`, and guide the testing and evidence collection. ChatGPT also helped draft this report using the observed experimental results.

I ran the commands in GitHub Codespaces, checked the outputs, and verified the implementation through the recorded tests.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | Continuous packet processing before adding the hop limit |
| `evidence/delivery.txt` | Successful tunnel delivery and ordinary IPv4 forwarding |
| `evidence/loop-after.txt` | Forwarding loop terminating with hop-limit protection |
| `evidence/mixed.txt` | Mixed-version delivery, IPv4 forwarding, and loop testing |
| `evidence/h2-delivery.txt` | Receiver log for h2 |
| `evidence/h3-delivery.txt` | Receiver log for h3 |