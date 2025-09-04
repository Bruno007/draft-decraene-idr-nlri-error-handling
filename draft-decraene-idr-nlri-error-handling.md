---

stand_alone: true

ipr: trust200902

cat: std

submissiontype: IETF

area: rtg

wg: Internet Engineering Task Force

updates: 7606

docname: draft-decraene-idr-nlri-error-handling-00


title: NLRI Error handling



lang: en

kw:

  - bgp
  - error-handling
  - NLRI

author:


  name: Bruno Decraene

  org: Orange

  email: bruno.decraene@orange.com


  name: John G. Scudder

  org: HPE

  email: jgs@juniper.net


normative:

  RFC2119:
  RFC4271:
  RFC4760:
  RFC7606:
  RFC8174:  

informative:

  RFC3107:
  RFC8277:  
  I-D.ietf-idr-bgp-car:
  I-D.ietf-idr-bgp-ct:

--- abstract

RFC 7606 partially revises the error handling for BGP UPDATE messages.
It reduces the cases of BGP session reset by defining and using less impacful error handling approaches such as attribute discard and treat-as-withdraw when applicable.
The treat-as-withdraw approach requires that the entire NLRI field of the MP_REACH_NLRI attribute be successfully parsed. This typically excludes parsing errors in this attribute.
This is exacerbated by the use non-key data within NLRI which introduces parsing complexity and additional error cases.

This specification defines a non-transitive BGP attribute, the "Treat-As-Withdraw Attribute" to encode NLRIs as per the format of MP_UNREACH_NLRI.
This attribute is used to allow the treat-as-withdraw error-handling approach in case an error in the MP_UNREACH_NLRI attribute prevents the parsing of its NLRIs.

--- middle

# Introduction {#intro}

According to the base BGP specification {{RFC4271}}, a BGP speaker that receives an UPDATE message containing a malformed attribute is required to reset the session over which the offending attribute was received.
This behavior is undesirable because a session reset impacts not only routes with the offending attribute but also other valid routes exchanged over the session.


{{RFC7606}} revise BGP error handling with the goal to minimize the impact on routing of a malformed UPDATE message while maintaining protocol correctness to the extent possible.
For most BGP attributes, a malformed attribute may be handled using attribute discard or Treat-as-withdraw.
Both approaches preserve the routing of all the NLRIs not advertised in this BGP UPDATE message.
However, as indicated in section 3 of {{RFC7606}}, the uses of treat-as-withdraw requires that the entire NLRI field of the MP_REACH_NLRI attribute be successfully parsed.
This typically excludes errors while parsing the MP_REACH_NLRI attribute.

{{RFC4760}} allows Border Gateway Protocol (BGP) to advertise general routing information in the Network Layer Reachability Information (NLRI) field of the UPDATE message.
Some specifications, such as {{RFC8277}}, {{I-D.ietf-idr-bgp-car}}, and {{I-D.ietf-idr-bgp-ct}} carry both a key field and a non-key field in the NLRI.
The key field is typically the real NLRI.
The non-key field carries extra data that is NLRI specific and hence not located in the BGP paths attributes for packing optimisation purpose.
For example, {{RFC8277}} carries the Prefix in the key field and one label (stack) in the non-key field.
As another example, {{I-D.ietf-idr-bgp-car}} defines a BGP CAR SAFI explicitly carrying Key Fields and Non-Key Fields as a list of TLVs.
In case of a BGP withdraw, the key is indicated in the MP_UNREACH_NLRI attribute to withdraw the unfeasible routes, while the non-key data is typically not encoded.

This specification defines a new BGP non-transitive attribute, the "Treat-As-Withdraw Attribute" to carry the NLRIs using the simple and existing format of MP_UNREACH_NLRI.
This attribute is used to allow the treat-as-withdraw error-handling approach in case an error in the MP_UNREACH_NLRI attribute is preventing the parsing of its NLRIs.

## Requirements Language

{::boilerplate bcp14-tagged}


# Treat-As-Withdraw Attribute {#format}


The Treat-As-Withdraw attribute is an optional, non-transitive BGP path attribute with type code TBD1. 
The format of the Treat-As-Withdraw attribute is the same as the format of the MP_UNREACH_NLRI as defined in section 4 of {{RFC4760}}.


# Sending the Treat-As-Withdraw Attribute {#sending}

The Treat-As-Withdraw attribute may be sent in any BGP UPDATE message carrying the MP_REACH_NLRI attribute.
It MUST NOT be sent in UPDATE message not carrying the MP_REACH_NLRI attribute.
To facilitate the determination of the NLRI field in an UPDATE message with a malformed attribute, the Treat-As-Withdraw SHALL be encoded as the very first path attribute in an UPDATE message, followed by the MP_REACH_NLRI attribute.


The Treat-As-Withdraw attribute is generally useful as its encoding is simpler than the encoding of the MP_REACH_NLRI hence it maximizes the chances of handling an error in the MP_REACH_NLRI attribute using the treat-as-withdraw approach.
It is specifically usuful for AFI/SAFI carrying non-key data in the NLRI such as {{RFC8277}}, {{I-D.ietf-idr-bgp-car}}, and {{I-D.ietf-idr-bgp-ct}} as these NLRI are longer and more complex, hence have a higher probability of error. In addition, in case of error they have a lower probability of being able to parse the full list of NLRIs.

# Receiving the Treat-As-Withdraw Attribute {#receiving}

In the case of an UPDATE message with a malformed MP_REACH_NLRI attribute and a correctly formed Treat-As-Withdraw attribute SHALL be handled using the approach of "treat-as-withdraw".
The UPDATE message SHALL be handled as if received with only the Treat-As-Withdraw attribute -all others attributes being ignored- and the Treat-As-Withdraw Attribute handled as a MP_UNREACH_NLRI attribute.


In the case of an UPDATE message with a correctly formed MP_REACH_NLRI attribute, the Treat-As-Withdraw attribute SHOULD be parsed and its lists of NLRI compared to the list of NLRI present in the MP_REACH_NLRI attribute.
In case of difference, the Treat-As-Withdraw attribute SHALL be ignored.
The reasoning is to fall back to the error handling pre-existing this document.
However, because this reveals an error in either the Treat-As-Withdraw attribute or the MP_REACH_NLRI attribute, a BGP speaker must provide debugging facilities to permit issues caused by a malformed attribute to be diagnosed.
At a minimum, such facilities must include logging an error listing the NLRI involved and containing the entire malformed UPDATE message when such an attribute is detected.
The malformed UPDATE message should be analyzed, and the root cause should be investigated.


A BGP speaker receiving a BGP UPDATE with a Treat-As-Withdraw attribute MUST remove this attribute from the UPDATE after having successfully parsed this BGP UPDATE.

# Attribute Error Handling {#error}

The Treat-As-Withdraw attribute has the same format than the MP_UNREACH_NLRI and hence have the same malformed conditions.
As per {{receiving}}, an UPDATE message with a malformed Treat-As-Withdraw attribute is handled using the approach of "attribute discard".

# IANA Considerations {#IANA}

IANA is requested to allocate a new optional, non-transitive attribute called "Treat-As-Withdraw" from the BGP Path Attributes registry of the Border Gateway Protocol (BGP) Parameters group.

| Value | Code | Reference |
+:------|:-----|:----------|
| TBD1  | Treat-As-Withdraw |(this doc) |




# Security Considerations {#Security}

The Treat-As-Withdraw attribute does not change BGP security considerations.
An attacker having the ability to send or modify a BGP Message has the ability to withdraw any NLRI, with or without the Treat-As-Withdraw attribute.


# Acknowledgements {#Acknowledgements}

{: numbered="false"}



