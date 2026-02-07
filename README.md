# Multiprotocol Label Switching (MPLS)

In this exercise you will see a different forwarding technique: Multiprotocol Label Switching. Your final objective will be to implement a basic version of MPLS forwarding in the data plane.

MPLS attaches labels to data packets to drive the packet forwarding decisions. That is, instead of forwarding packets based on their IP addresses, switches forward packets just by looking up the contents of the MPLS labels that are attached to packets.

As you will see, MPLS has numerous benefits such as the possibility of creating end-to-end circuits for all types of packets or an extensive support for traffic engineering.

### MPLS Terminology

Before we begin, we need some vocabulary to understand the concepts better:

- Multiprotocol Label Switching (MPLS): A highly scalable, data-carrying mechanism that is independent of any data link layer protocol.
- Label Edge Router (LER): A router that operates at the edges of an MPLS network. An LER determines and applies the appropriate labels and forwards the labeled packets into the MPLS domain.
- Label Switch Router (LSR): A router that switches the labels that are used to route packets through an MPLS network. You can understand LSRs as *all* the MPLS-capable switches in the network. LERs are also LSRs.
- Label Switched Path (LSP): A route through an MPLS network, defined by a signaling protocol such as the Border Gateway Protocol (BGP). The path is set up based on criteria in the forwarding equivalence class (FEC).
- Forwarding Equivalence Class (FEC): A set of packets with similar characteristics that might be bound to the same MPLS label. **An FEC tends to correspond to a label switched path (LSP); however, an LSP might be used for multiple FECs.**

### MPLS Label Encoding

In actual MPLS, the label stack is represented as a sequence of *label-stack entries*. Each label-stack entry is represented by 4 bytes:

```bash
Label: Label Value, 20 bits
Exp: Experimental Use, 3 bits
S: Bottom of Stack, 1 bit
TTL: Time to Live, 8 bits
```

The meaning and usage of each field is as follows:

> Label Value: This 20-bit field carries the actual value of the label. When a labeled packet is received, the label value at the top of the stack is looked up. As a result of a successful lookup one learns:
>  * the next hop to which the packet is to be forwarded
>  * the operation to be performed on the label stack before forwarding; this operation may be to replace the top label stack entry with another, or to pop an entry off the label stack, or to replace the top label stack entry and then to push one or more additional entries on the label stack.

> Experimental Use: This three-bit field is reserved for experimental use.

> Bottom of Stack (S): This bit is set to one for the last entry in the label stack (i.e., for the bottom of the stack), and zero for all other label stack entries.

> Time to Live (TTL): This eight-bit field is used to encode a time-to-live value.

The label-stack entries appear **_after_** the data-link layer headers, but **_before_** any network layer headers. The top of the label stack appears **_earliest_** in the packet, and the bottom appears **_latest_**. The network-layer packet immediately follows the label-stack entry which has the `S` bit set.

</br>

## 1. Introduction to MPLS

<img src="images/icons8-idea-32.png"> As a packet travels from one router to the next, each router makes an independent forwarding decision for that packet. Choosing the next hop can be thought of as the composition of two functions. The first function partitions the entire set of possible packets into a set of "Forwarding Equivalence Classes (FECs)". The second maps each FEC to a next hop. Insofar, as the forwarding decision is concerned, different packets which get mapped into the same FEC are indistinguishable.

In MPLS, the assignment of an arbitrary packet to a specific FEC is done just once: as the packets enter the MPLS network. The FEC to which the packet is assigned is encoded as a short fixed length value known as a `label`. When a packet is forwarded to its next hop, the label is sent along with it; that is, the packets are "labeled" before they are forwarded into the network.

In the MPLS forwarding paradigm, once a packet is assigned to a FEC, no further header analysis is done by subsequent routers; all forwarding is driven by the labels.

MPLS mainly relies on three features:

1) A Match-Action Table that maps classes of packets into the desired labels (i.e., FTN).
2) Labels that are carried by the packets.
3) A Match-Action Table that maps labels to egress ports to execute the forwarding (i.e., the NHLFE).

>The **FEC-to-NHLFE Map (FTN)** points each FEC to a NHLFE (or a set of NHLFEs when Traffic Engineering is applied). It is used when forwarding packets that arrive unlabeled, yet to be labeled before being forwarded.

>A **label** is a short, fixed-length, locally significant identifier which is used to identify a FEC. It is the responsibility of each Label Switching Router (LSR) to ensure that it can uniquely interpret its incoming labels.

>**Next Hop Label Forwarding Entry (NHLFE)** is used when forwarding a labeled packet. It contains information about the packet's next hop.

</br>

## Exercise: Implementing the MPLS Basics

<img src="images/icons8-job-32.png"> In this exercise, your task will be to implement a mock MPLS architecture. In this mock architecture, we will have a single label for each packet. These labels will be imposed by the ingress LER and they will hold a global meaning. In short, this mock MPLS architecture encodes FECs into labels.

### Before starting

We provide you some files that will help you through the exercise.

- `topology.json`: describes the topology that you will use throughout the exercise.
- `mpls.p4`: contains the p4 program skeleton that you will use as a starting point for this exercise (mock MPLS).

**Note**: This time you will not be able to run `make run` until you finish some of the `TODOs`.

In this exercise, you will work with the following topology. Your objective is to enable communication from h1 to h2 and h3, by implementing different functionalities of MPLS.

<p align="center">
<img src="images/mpls.png" title="MPLS Topology">
<p/>

### Task 1. Adding MPLS-label support

1. Create a new `etherType` to indicate the MPLS protocol. LERs will use this `etherType` to detect MPLS packets. Just as `TYPE_IPv4`, `TYPE_MPLS` will have a size of 16 bits. You can pick the value that you want. If you want to follow the standard use `0x8847`.

2. Create a header type for the MPLS label. The header should have 4 fields as described [above](#mpls-label-encoding). Each layer in the header stack should have a 20-bit field called `label`, a 3-bit field called `exp`, a 1-bit field called `s` (bottom-of-stack), and an 8-bit field called `ttl`.

3. Instantiate the struct `headers` including the MPLS header that you have just created. Note that the MPLS header will go in between the IPv4 header and the Ethernet header.

4. Create a parser, `parse_mpls`, that extracts the MPLS header whenever an MPLS packet is received.

5. Call the parser in case an MPLS packet is detected. You can use the constant `TYPE_MPLS` that you have created to know if the Ethernet packet contains an MPLS header, or just the IPv4.

### Task 2. Setting packet labels from ingress switches (s1)

Implement the functionality that allows the entry switch (s1) to add an extra header in the packet that will be used as label across the path.

First, you need to identify packets accessing the MPLS network. To that end:

1. Note that we have created a metadata field `is_ingress_border`, that will help us identify the packets that are processed from an **ingress switch** (i.e., the switch through which the packet **accesses the MPLS network**).

2. Create a new table `check_is_ingress_border` that checks if the ingress port of the packet corresponds to an entry of the MPLS network. The table should (exact-) match on the `standard_metadata.ingress_port`, and should execute the action `set_is_ingress_border` in case it is hit.

3. Create the action `set_is_ingress_border` that sets the `is_ingress_border` metadata field to 1 whenever executed.

4. If you take a look at `s1-runtime.json`, you can see how we have already filled the `check_is_ingress_border` table entries for you. In this case (switch s1), all packets entering from port 1 (i.e., coming from h1) will hit the entry, and therefore will be identified.

```
    {
      "table": "MyIngress.check_is_ingress_border",
      "match": {
        "standard_metadata.ingress_port": [
          1
        ]
      },
      "action_name": "MyIngress.set_is_ingress_border",
      "action_params": {}
    }
```

Switch s1 will have to act as an ingress_border switch for those packets, and add an MPLS header to them, selecting the best label according to their forwarding equivalency class (FEC). Implementing this functionality will be your next task.

We already give you the code for the ingress pipeline. Take a look at it and make sure you understand the logic.
```
apply {

  // We check whether it is an ingress border port
  check_is_ingress_border.apply();

  if(meta.is_ingress_border == 1){

      // We need to check whether the header is valid (as mpls label is based on dst ip)
      if(hdr.ipv4.isValid()){

          // We add the label based on the destination
          fec_to_label.apply();
      }
  }
```

As you can see, the code just checks whether the switch should act as an ingress switch and, in case it should, the table `fec_to_label` is applied. This table will be responsible for selecting, and adding, the MPLS label to the packet.

**Which MPLS label will we add?** We want to create two forwarding equivalency classes (FECs): one for packets destined to h2 (10.7.2.2), and one for packets destined to h3 (10.7.3.2). In order to do that:

5. Create the `fec_to_label` table. It should (lpm-) match on the `hdr.ipv4.dstAddr`, and should execute an action `add_mpls_header` in case it is hit, passing as parameter the label corresponding to the FEC to which it has matched. You should use two different label values to identify packets for each of the two FECs.  In `s1-runtime.json`, we use labels 2 and 3 as follows:

```
    {      
      "table": "MyIngress.fec_to_label",
      "match": {
        "hdr.ipv4.dstAddr": [
          "10.7.2.2",
          32
        ]
      },
      "action_name": "MyIngress.add_mpls_header",
      "action_params": {
          "tag": 2
      }
    },
    {      
      "table": "MyIngress.fec_to_label",
      "match": {
        "hdr.ipv4.dstAddr": [
          "10.7.3.2",
          32
        ]
      },
      "action_name": "MyIngress.add_mpls_header",
      "action_params": {
          "tag": 3
      }
    }
```

⚠️ Note that the IP assignment strategy used here is different from the one from previous exercises.

1. At this point the ingress switch knows how to add MPLS labels based on the destination IP. However, it still does not know to which port to send packets. Let's add that functionality!

### Task 3. Forwarding packets based on labels

Implement the functionality that allows switches to (i) read the label from the packet header, and (ii) use it to forward the packets accordingly. To that end:

1. Uncomment the last lines of the ingress pipeline. Note that the command `isValid()` is used to check whether the processed packet contains an MPLS header. If that is the case, then the label is read and the packet is forwarded accordingly.

```
// We select the egress port based on the mpls label
if(hdr.mpls.isValid()){
    mpls_tbl.apply();
}
```

2. Create the `mpls_tbl` table, that (exact-) matches on the label field in the `mpls` header, and calls the action `mpls_forward` which sets the mac addresses and egress port accordingly. We already give you the action, and the table entries for `s1` as example. Note that `mac` addresses between switches do not really matter since they are not checked, but when the packet is sent to a real host, the `mac` has to match. If that is not the case, packets will be dropped and not reach the application layer of e.g., `ping` or `iperf`.

```
    {
      "table": "MyIngress.mpls_tbl",
      "match": {
        "hdr.mpls.label": [
          2
        ]
      },
      "action_name": "MyIngress.mpls_forward",
      "action_params": {
        "dstAddr": "00:00:00:02:01:00",
        "port": 2
      }
    },
    {
      "table": "MyIngress.mpls_tbl",
      "match": {
        "hdr.mpls.label": [
          3
        ]
      },
      "action_name": "MyIngress.mpls_forward",
      "action_params": {
        "dstAddr": "00:00:00:03:01:00",
        "port": 3
      }
    }
```

1. We also fill up the corresponding `sx-runtime.json` entries for the other switches in the path. Such that packets follow the paths indicated in the topology figure. FEC corresponding to packets destined to h2 should follow the blue path, while packets destined to h3 should follow the red path.

2. To be able to ping between `h2` and `h3` we need to add normal `ipv4` forwarding capabilities to the switches. For that, create an `ipv4_lpm` table that matches on `ipv4.dstAddr` (with an lpm match), and calls the action `ipv4_forward`. Finally, uncomment the lines of code that execute this table from the `apply` function in the ingress pipeline (look for TODO 3.4).

3. Create the `ipv4_forward` action that is called when using the normal `ipv4` forwarding. The action takes as parameter a destination mac address and an egress port. In this action, set the src ethernet address as the previous destination (mac swap), and use the action parameter to set the new destination ethernet address. Set the egress port, and decrease the TTL.

4. As you modified the TTL of the IP header, and the IP header has a checksum field, it needs to be recomputed. We already provide you the implementation. You just have to uncomment the code and understand what it does.

5. Use `tcpdump` or the pcap log files at the output of each switch to verify that your forwarding behavior works as expected (`tcpdump -enn -l -i <interface_name>`). To this point, your packets should be arriving to `s7`. 

### Task 4. Removing packet labels from egress switches (s7)

Extend the functionality that allows the last switch (`s7`) to remove the packet header once the forwarding has been decided, deparse the packet, and forward.

In the same way that you have been able to identify switches that should act as ingress to the MPLS network, now you have to identify egress switches. Note that we have created a metadata field `is_egress_border`, that will help us identify the packets that are processed from an **egress switch** (i.e., the switch through which the packet **leaves the MPLS network**).

1. Create a table `check_is_egress_border` that (exact-) matches on the `standard_metadata.egress_port` and executes the action `is_egress_border()` if it is hit. Uncomment line check_is_egress_border.apply().


`standard_metadata.egress_port` is a read-only metadata field that can be accessed only from the egress pipeline, and which the traffic manager uses to tell the egress pipeline which egress port it has selected for forwarding. It should not be confused with `standard_metadata.egress_spec`, which is written by the P4 programmer at the ingress pipeline to tell the traffic manager which egress port should be selected for forwarding.

2. Take a look at what the action `is_egress_border()` is doing.

```
action is_egress_border(){
    hdr.mpls.setInvalid();
    hdr.ethernet.etherType = TYPE_IPV4;
    hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
}
```

As you might have guessed, it is removing the MPLS header. In p4, a header is removed is by setting it to invalid with the command `setInvalid()`. When the packet is deparsed, the deparser will see that the header is not valid and it will not emit it. Also note the importance of setting the EtherType back to `TYPE_IPV4`, so that future routers outside the MPLS network can correctly parse the packets. At this point, the packet exiting the MPLS network should be exactly the same as the one which entered it (i.e., all the packet processing executed by the MPLS network is completely transparent from the outside).

3. At this point you should already be able to send ping requests from h1 to h2 and h3. You can run `tcpdump` from each of the two receiving hosts to see the result: (`tcpdump -enn -l -i <interface_name>`)

<p align="center">
<img src="images/ping-h1.png">
<p/>

<p align="center">
<img src="images/tcpdump-h2.png">
<p/>

Although the ping request is arriving to both h2 and h3, their respective ping responses are still not getting back to h1.

### Task 5. Enabling bidirectional communication

It is your task now to modify your existing solution such that the ping responses from h2 and h3 get back to h1. To this end:

1. First, let replies from both h2 and h3 follow the blue path: `s7->s6->s4->s2->s1->h1`. To do that, create a FEC for all packets with destination h1 (10.1.1.2). You can use a new label (e.g., 1) to represent such traffic along all the switches. Note that you do not need to change anything from the p4-code itself. The only thing you have to do is to add more table entries in the `sx-runtime.json` files. Don't forget to also configure s7 as ingress border switch, for packets coming from ingress ports 3 and 4, and s1 as egress border switch for packets exiting from egress 1. 

2. Test that your solution is working by "pinging" h2 and h3 from h1. You should see the responses arriving correctly back to h1, and a 0% of packet loss, as shown in the picture below. Feel free to do some further test your implementation with other tools such as `iperf`. If you implemented the `ipv4_lpm` that allows `h2` to talk with `h3` you can test your solution with `pingall` which will test if every host can ping all the others.

<p align="center">
<img src="images/pingall.png">
<p/>




