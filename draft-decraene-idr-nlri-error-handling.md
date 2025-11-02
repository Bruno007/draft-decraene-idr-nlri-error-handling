---

stand_alone: true

ipr: trust200902

cat: std

submissiontype: IETF

area: rtg

wg: Internet Engineering Task Force

updates: 7606

docname: draft-decraene-idr-nlri-error-handling-01

title: The Key List BGP Attribute for NLRI Error handling

lang: en

kw:

  - bgp
  - error-handling
  - NLRI

author:

 -
  name: Bruno Decraene

  org: Orange

  email: bruno.decraene@orange.com

 -
  name: John G. Scudder

  org: HPE

  email: jgs@bgp.nu

normative:

  RFC2119:
  RFC4271:
  RFC4760:
  RFC5492:
  RFC7606:
  RFC8174:  

informative:

  RFC3107:
  RFC8277:  
  I-D.ietf-idr-bgp-car:
  I-D.ietf-idr-bgp-ct:

--- abstract

RFC 7606 partially revises the error handling for BGP UPDATE messages.
It reduces the cases of BGP session reset by defining and using less impactful error handling approaches, such as attribute discard and treat-as-withdraw when applicable.
The treat-as-withdraw approach requires that the entire NLRI field of the MP_REACH_NLRI attribute be successfully parsed. This typically means parsing errors in MP_REACH_NLRI cannot be handled by any means short of session reset.
This is exacerbated by the use of non-key data within NLRI, which introduces parsing complexity and additional error cases.

This specification defines a non-transitive BGP attribute, the "NLRI_KEY_LIST attribute", to encode NLRIs as per the format of MP_UNREACH_NLRI.
This attribute is used to allow the treat-as-withdraw error-handling approach to be used in case an error in the MP_REACH_NLRI attribute prevents the parsing of its NLRIs.

This document updates RFC 7606 by mandating that the NLRI_KEY_LIST attribute appear before the MP_REACH_NLRI (or any other) attribute in an UPDATE message.
--- middle

# Introduction {#intro}

According to the base BGP specification {{RFC4271}}, a BGP speaker that receives an UPDATE message containing a malformed attribute is required to reset the session over which the offending attribute was received.
This behavior is undesirable because a session reset impacts not only routes with the offending attribute but also other valid routes exchanged over the session.

{{RFC7606}} revises BGP error handling, with the goal of minimizing the impact on routing of a malformed UPDATE message, while maintaining protocol correctness to the extent possible.
For most BGP attributes, a malformed attribute may be handled using attribute discard or treat-as-withdraw.
Both approaches preserve the routing of all the NLRIs not advertised in the affected BGP UPDATE message.
However, as indicated in Section 3 of {{RFC7606}}, treat-as-withdraw can only be used if the entire NLRI field of the MP_REACH_NLRI attribute is successfully parsed.
This typically means parsing errors in MP_REACH_NLRI cannot be handled by any means short of session reset.

{{RFC4760}} allows the Border Gateway Protocol (BGP) to advertise general routing information in the Network Layer Reachability Information (NLRI) field of the UPDATE message.
Some specifications, such as {{RFC8277}}, {{I-D.ietf-idr-bgp-car}}, and {{I-D.ietf-idr-bgp-ct}} carry both a key field and a non-key field in the NLRI.
The key field is typically the real NLRI.
The non-key field carries extra data that is NLRI-specific and hence not located in the BGP path attributes for packing optimization purposes.
For example, {{RFC8277}} carries the Prefix in the key field and one label (stack) in the non-key field.
As another example, {{I-D.ietf-idr-bgp-car}} defines a BGP CAR SAFI explicitly carrying Key Fields and Non-Key Fields as a list of TLVs.
In case of a BGP withdraw, the key is indicated in the MP_UNREACH_NLRI attribute to withdraw the unfeasible routes, while the non-key data is typically not encoded.

This specification defines a new BGP non-transitive attribute, the "NLRI_KEY_LIST attribute", to carry the NLRIs using the simple and existing format of MP_UNREACH_NLRI.
Its most important use is for AFI/SAFI whose NLRI are considered to be at elevated risk of malformation. An example are AFI/SAFI that encode both a key field and a non-key field in the NLRIs of the MP_REACH_NLRI attribute, while encodes NLRI in a simpler way in the MP_UNREACH_NLRI attrbiute, e.g., with only the key field.
For such AFI/SAFI, in case of an error in the MP_UNREACH_NLRI attribute preventing the identification of all NLRIs key, the parsing of the NLRI_KEY_LIST attribute is more likely allow the identification of all NLRIs key.
This attribute is used to allow the treat-as-withdraw error-handling approach to be used when there is an error in the MP_REACH_NLRI attribute that prevents the parsing of its NLRIs.

## Requirements Language

{::boilerplate bcp14-tagged}

# NLRI_KEY_LIST Capability {#capability}

To avoid the overhead of sending and receiving an attribute which is not understood by the BGP speaker receiving it, this document defines a new BGP Capability {{RFC5492}} "NLRI_KEY_LIST", of type TBD2, and of length zero.
A BGP speaker that supports the reception of the NLRI_KEY_LIST attribute ({{receiving}}) SHOULD advertise the NLRI_KEY_LIST Capability Advertisements.
A BGP speaker SHOULD NOT send the NLRI_KEY_LIST attribute unless its peer has advertised the NLRI_KEY_LIST Capability. 
Note however that if the attribute is sent, it will cause no harm (an incapable implementation will disregard the attribute, per the base BGP specification). A potential reason to send the attribute to a peer that has not advertised support is to avoid fragmenting a peer group. 

# NLRI_KEY_LIST Attribute 

## NLRI_KEY_LIST Format {#format}

The NLRI_KEY_LIST attribute is an optional, non-transitive BGP path attribute with type code TBD1. 
The format of the NLRI_KEY_LIST attribute is the same as the format of the MP_UNREACH_NLRI as defined in Section 4 of {{RFC4760}} and the relevant specification for the AFI/SAFI in question.

## Sending the NLRI_KEY_LIST attribute {#sending}

The NLRI_KEY_LIST attribute may be sent in a BGP UPDATE message carrying the MP_REACH_NLRI attribute.
It MUST NOT be sent in an UPDATE message not carrying the MP_REACH_NLRI attribute.
To facilitate the determination of the NLRI key list in an UPDATE message with a malformed attribute, the NLRI_KEY_LIST SHALL be encoded as the very first path attribute in an UPDATE message, followed by the MP_REACH_NLRI attribute. (This represents an update to Section 5.1 {{RFC7606}}, which mandated that the MP_REACH_NLRI come first.)
TThe list and the ordering of NLRIs within the NLRI_KEY_LIST MUST be the same as the list and their ordering within the corresponding MP_REACH_NLRI. <!-- This is only needed if the requirement to compare the TAW to the MP_REACH_NLRI during normal operation is retained. If we end up deciding on the other approach instead, i.e., ignore the TAW unless there is an error, this ordering requirement should be removed. The reasoning for imposing the ordering requirement is, if we're going to always compare them, we are adding work to the inner loop of the protocol, for every update, so we should minimize that work. Requiring identical ordering means the comparison can be guaranteed to be O(N) in the number of NLRI; if ordering isn't required, the worst-case is more like O(N^2) or in any case not as good as O(N). -->

If the AFI/SAFI specification allows for different NLRI encodings in the MP_UNREACH_NLRI, the sender MUST use the simplest encoding. The receiver MUST accept any valid encoding. For example, {{RFC3107}} allows the use of either the MPLS label stack originally sent or the static 0x800000 value. The latter is simpler in that the size is smaller, fixed, and the number of labels to parse is minimized.

The NLRI_KEY_LIST attribute is generally useful as its encoding is simpler than the encoding of the MP_REACH_NLRI, hence it maximizes the chances of handling an error in the MP_REACH_NLRI attribute using the treat-as-withdraw approach.
In particular the NLRI_KEY_LIST attribute does not carry the variable length "Network Address of Next Hop" field nor the "Length of Next Hop Network Address" which, if erroneous, trigger a BGP session reset as per {{RFC7606}}. 
<!-- Furthermore, in some implementations, it may be the case that a different code path is used to generate the MP_UNREACH_NLRI encoding than is used to generate the MP_REACH_NLRI encoding. This can be seen as beneficial, analogous to "it's ideal if redundant parts come from different suppliers". -->
It is notably, although not exclusively, useful for AFI/SAFI carrying non-key data in the NLRI such as {{RFC8277}}, {{I-D.ietf-idr-bgp-car}}, and {{I-D.ietf-idr-bgp-ct}} as these NLRI are longer and more complex, hence have a higher probability of error. In addition, in case of error, they have a lower probability of being able to parse the full list of NLRIs.
It is less useful when the NLRI encoding is the same for MP_REACH_NLRI and MP_UNREACH_NLRI.

## Receiving the NLRI_KEY_LIST attribute {#receiving}

An UPDATE message with a malformed MP_REACH_NLRI attribute and a correctly formed NLRI_KEY_LIST attribute SHALL be handled using the approach of "treat-as-withdraw".
The UPDATE message SHALL be handled as if received with only the NLRI_KEY_LIST attribute - all other attributes being ignored - and the NLRI_KEY_LIST attribute handled as an MP_UNREACH_NLRI attribute.

In the case of an UPDATE message with a correctly formed MP_REACH_NLRI attribute, the NLRI_KEY_LIST attribute SHOULD be parsed and its list of NLRI compared to the list of NLRI present in the MP_REACH_NLRI attribute.
In case of difference, the NLRI_KEY_LIST attribute SHALL be ignored.
However, because this reveals an error in either the NLRI_KEY_LIST attribute or the MP_REACH_NLRI attribute, a BGP speaker must provide debugging facilities to permit issues caused by a malformed attribute to be diagnosed.
At a minimum, such facilities must include logging an error listing the NLRI involved and containing the entire malformed UPDATE message when such an attribute is detected.
The malformed UPDATE message should be analyzed, and the root cause should be investigated.

<!-- When a BGP speaker receives a BGP route that includes the NLRI_KEY_LIST attribute, it MUST discard the NLRI_KEY_LIST attribute after having successfully parsed this BGP UPDATE. -->

## NLRI_KEY_LIST attribute Error Handling {#error}

The NLRI_KEY_LIST attribute has the same format as the MP_UNREACH_NLRI and hence has the same conditions under which it is considered malformed.
As per {{receiving}}, an UPDATE message with a malformed NLRI_KEY_LIST attribute is handled using the approach of "attribute discard".

# Operational Considerations

<!-- This is arguably not best called "Operational Considerations". It could also be a subsection under "Sending the NLRI_KEY_LIST attribute", or even part of the main body of that section (but then it gets kind of long and wordy). -->

The choice of whether or not to send the NLRI_KEY_LIST attribute is up to the implementor and the operator. As is discussed elsewhere in this document, the attribute is considered more likely to be valuable for AFI/SAFI with more complex NLRI encodings, and less likely to be valuable in the case where the encodings used for the MP_REACH_NLRI and MP_UNREACH_NLRI are the same.

Drawbacks of sending the attribute include space overhead in the UPDATE message, as well as time overhead to form the attribute on the sender side and to validate it on the receiver side. 

The primary advantage of sending the attribute is avoidance of session reset with concomitant service disruption, but one may also observe the potential to detect non-fatal errors which would otherwise be invisible, when the NLRI_KEY_LIST attribute is compared to the MP_REACH_NLRI attribute. The latter might motivate an operator to configure support even for less complex AFI/SAFI.

An implementation SHOULD provide a configuration option allowing the operator to send, or not send, the attribute with any AFI/SAFI. The default may differ for different AFI/SAFI; this specification does not, in any case, mandate a default. 

# IANA Considerations {#IANA}

IANA is requested to allocate a new optional, non-transitive attribute called "NLRI_KEY_LIST" from the BGP Path Attributes registry of the Border Gateway Protocol (BGP) Parameters group.

| Value | Code | Reference |
+:------|:-----|:----------|
| TBD1  | NLRI_KEY_LIST |(this doc) |

IANA is requested to allocate a new capability called "NLRI_KEY_LIST" from the Capability Codes registry.

| Value | Description | Reference |
+:------|:-----|:----------|
| TBD2  | NLRI_KEY_LIST |(this doc) |

# Security Considerations {#Security}

The NLRI_KEY_LIST attribute does not change BGP security considerations.
An attacker having the ability to send or modify a BGP message has the ability to withdraw any NLRI, with or without the NLRI_KEY_LIST attribute.

# Acknowledgements {#Acknowledgements}

The authors of this specification thank Jeffrey Haas and Robert Raszuk for their review and comments.

{: numbered="false"}

