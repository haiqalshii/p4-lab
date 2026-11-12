# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **1995**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

I created the tunnel loop by modifying the entry for tunnel destination 3 on
s2. The entry had handle `0x2` and originally forwarded through port 3. I
changed its action parameter to port 1 with:

`sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"`

Switch s1 already forwarded tunnel destination 3 towards s2, while the modified
s2 entry sent it back towards s1. Therefore, the same packet travelled repeatedly
between the two switches.

In `evidence/loop-before.txt`, both switches continued processing the packet
throughout the observation period. The processing counts increased between
successive measurements, showing that the packet remained in the loop and
would not stop without intervention.

## 2. Design: the hop limit


The tunnel hop limit as an unsigned 8-bit field placed after
`dst_id`. The sender starts it at 8. At every switch, ingress processing
checks the value before tunnel forwarding. A packet with zero remaining
hops is dropped; otherwise, the value is decreased by one and
`myTunnel_exact` is applied. Checking zero first prevents subtraction
from wrapping zero around to 255. The parser, deparser, sender and
receiver will all use the same updated header format.


## 3. Implementation

I extended the P4 and Scapy tunnel formats with the same 8-bit
`hop_limit` field. The ingress control drops a tunnelled packet when
the value is zero; otherwise, it subtracts one before applying the
tunnel table.

## 4. Evidence: it works


The results in `evidence/delivery.txt` show that a tunnelled packet with
destination 2 reached h2 and a packet with destination 3 reached h3.
Both packets started with a hop limit of 8 and arrived with 6 because
they were processed by s1 and s2. The same evidence file shows that a
plain ping from h1 to h3 still succeeded, so the IPv4 forwarding path
was not broken by the tunnel change.

I recreated the original loop by changing s2's tunnel-destination-3
entry to output port 1. In `evidence/loop-after.txt`, the first count
showed 133 processing events on s1 and 132 on s2. The
counts were unchanged five seconds later, showing that the packet had
stopped circulating.

The result matches my design. A starting limit of 8 permitted eight
switch forwards. The packet then arrived at the next switch with zero
remaining hops and was dropped. The final drop is also counted as a
packet-processing event, so the total processing count can be one more
than the number of successful forwards.

## 5. Evaluation

Before running the rollout, I predicted that s2 would correctly
read `proto_id` and `dst_id` because the new field was appended after
them. However, its old parser expects IPv4 to begin immediately after
`dst_id`, so it would treat the new hop-limit byte as the first byte of
the IPv4 header. 

Before testing, I predicted that old `s2` would correctly read
`proto_id` and `dst_id`, but would mistake the appended hop-limit byte
for the start of IPv4. I also expected the mixed-version loop to stop,
but more slowly because only new `s1` decreases the hop limit.

The frame reached `h2`, but `receive.py` could not decode its UDP
payload. The capture in `evidence/mixed.txt` contains
`0800 0002 07 45`: the protocol ID, destination ID, remaining hop
limit, and actual IPv4 start. Old `s2` parsed `07` as the first IPv4
byte, shifting the header and causing checksum recomputation to change
the real protocol value from UDP (`0x11`) to `0x0d`. This shows that
preserving existing field positions is insufficient when the header
length changes.

### Safe rollout

A safe rollout could use a new EtherType for the new header. Switches
would first be upgraded to understand both formats while senders
continued using the old one. After every transit switch was verified,
senders could enable the new format and the old format could later be
removed. This uses additional parser resources temporarily but avoids
silently misparsing packets.

Another option is a coordinated upgrade where all switches are updated
before any sender uses the new header. This is simpler but requires more
coordination and possibly a maintenance window. Adding a version field
now would not fix compatibility because the existing old parser was not
designed to read it.

### Data plane versus control plane

The hop limit provides data-plane containment for packets affected by
runtime loops, stale entries or partial upgrades. It cannot prevent the
configuration error, but limits its effect. Week 3's `check_paths()`
provides control-plane prevention by finding loops and missing routes in
the intended configuration before deployment. It cannot detect every
difference between the model and the running switches. Real networks
use both prevention and runtime containment.

### Hardware cost

The change adds one byte to each tunnelled packet. It also requires the
parser and deparser to handle another field and ingress to perform a
comparison, subtraction and conditional drop for every tunnelled
packet. These operations consume fixed pipeline resources and could
require another stage if the current stage is full. The design does not
need per-flow state or another lookup table.

## AI use

I used OpenAI Codex/ChatGPT throughout this assignment. It explained the
C1–C6 requirements, helped interpret BMv2 table handles, switch logs and the
mixed-version hexadecimal packet capture, and suggested the 8-bit hop-limit
design with a default value of eight. It provided line-by-line suggestions for
changes to `tunnel.p4`, `send.py` and `receive.py`, including the decrement/drop
logic, the `--hops` option and receiver output. It also corrected indentation
in `receive.py`, suggested commands for collecting the required evidence, and
helped edit the report and pull-request description.

I applied the changes in my repository, ran all Mininet and mixed-rollout
experiments myself, checked the actual outputs, and revised the explanations
to match my evidence. I can explain the submitted code, design decisions and
test results.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
