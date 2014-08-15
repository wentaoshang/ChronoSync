===============
Design Document
===============

Terminology
===========

User
  an user which participates in the synchronization.
  An user is uniquely identified by a namespace, e.g., ``/ndn/ucla/alice``.

Session
  an instance of a user in the synchronization group.
  A session is uniquely identified by the namespace of the user and a sessionId:

    /<user_namespace>/[SessionId]

  The sessionId is encoded as `NDN-TLV nonNegativeInteger
  <http://named-data.net/doc/ndn-tlv/tlv.html#non-negative-integer-encoding>`_.
  A user can publish the data packets to synchronize under the session namespace.

  A user may have more than one sessions in a synchronization group.
  For example, one may join the synchronization through two sessions,
  one from his mobile phone and the other from his laptop.
  Each session is handled independently, even when two sessions belong to the same user.

Entity
  the process who runs a session.
  It could be a chatroom process running on end host machine.

Sequence Number
  Data packets published in the same session are indexed by continuous sequence numbers starting
  from 0.

    /<user_namespace>/[SessionId]/[SeqNo]

  The sequence number is encoded as `NDN-TLV nonNegativeInteger
  <http://named-data.net/doc/ndn-tlv/tlv.html#non-negative-integer-encoding>`_.
  The latest status of a session is represented by its largest sequence number.

Synchronization Group
  consists of several sessions which synchronize their latest status with each other.
  A sync group has its own namespace under which sessions publish their status updates.
  The namespace is constructed by concatenating a multicast prefix and the group name
  (as one component).
  For now, the multicast prefix is temporarily defined as

    /ndn/broadcast/[AppName]

  e.g., ``/ndn/broadcast/ChronoChat``.
  And a sync group name is defined as

    /ndn/broadcast/[AppName]/[GroupName]

  e.g., ``/ndn/broadcast/ChronoChat/letschat``.
  It is recommended to make group name unique.
  When two sync groups accidentally use the same group name,
  they may interfere each other's synchronization process.
  Eventually, this problem should not exist when sync-interests can be authenticated.

Data Structure
==============

Sync Tree
  a data structure which represents the knowledge of the sync group.

A sync tree has two levels: root and leaves.
Each leaf represents a session and is supposed to contain the latest sequence number of the session.
Leaves are ordered according to `Canonical Order
<http://named-data.net/doc/ndn-tlv/name.html#canonical-order>`_).

The sync tree is summarized by a root digest.
Suppose that *n* leaves are canonically ordered and
that the session name and sequence number of the *i*-th node is denoted by *N*\ :sub:`i`\  and
*S*\ :sub:`i`\, the root digest is calculated as:

    digest = H(H(N\ :sub:`0`\  | S\ :sub:`0`\ ) | H(N\ :sub:`1`\  | S\ :sub:`1`\ ) | ... | H(N\ :sub:`n-1`\  | S\ :sub:`n-1`\ ))

Entities synchronize the their knowledge about the sync group in terms of exchanging the root digest
of their sync tree.

Security Assumption
===================

We first assume that routing prevents malicious attacker from the multicast group,
i.e., malicious attackers cannot receive sync-interests.

Trust model is left to the upper layer applications.

Knowledge Synchronization
=========================

Normal Case
-----------

An entity keeps an outstanding **sync-interest** which carries the current root digest of its sync
tree:

    /ndn/broadcast/[AppName]/[GroupName]/[Digest]

With SHA-256 hash function, the digest is a 32-byte name component.
When the tree is empty, the digest is
``e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855``,
which is a digest with empty input.

The sync-interest will multicast to all entities in the sync group.
When every entity has the same knowledge, the root digest of their sync trees should be the same.
The sync-interests will be aggregated in the network, and their forwarding states in routers
constitute a multicast tree.

Once an entity changes its status (i.e., sequence number is updated), it should generate a data
packet, called **sync-reply**, which contains its session name and the latest sequence number.
The sync-reply will be forwarded back to all the entities.
These entities will update their knowledge set and the root digest of their sync trees,
and express a sync-interest with the new root digest.

The name of a sync-reply extends the sync-interest name by one name component ``Nonce``.
In case that the actual size of a sync-reply exceeds the network layer packet size limit,
a sync-reply needs to be broken into multiple segments.
Any pair of {prefix, seqNo} should not be splited into two segments.
Therefore, the sync-reply name could be:

    /<sync-interest>/[Nonce]/[SegmentNo]

``SegmentNo`` follows the definition specified in `NDN naming convention
<http://named-data.net/wp-content/uploads/2014/08/ndn-tr-22-ndn-memo-naming-conventions.pdf>`__.



Recovery
--------

An entity may receive a sync-interest with a digest that is different from the entity's current
root digest.
The received digest could fall into either of two cases: old digest or unknown digest.

To tell whether the received digest is an old one,
each entity should keep a log about how its root digest changes over time.
If the received digest can be found from the log, it must be the old digest.
With the log, the receiving entity can tell which sessions have changed the states since the state
represented by the old digest, therefore it can put the latest sequence number of these sessions
into sync-reply.
Therefore the size of sync-reply is roughly determined by the number of sessions in a sync group.

Unrecognized digest will happen when both sender and receiver of the sync-interest have some state
updates that the other side does not have
(e.g., two sessions generate state updates at the same time;
or an entity has not received some state updates yet).
When receiving an unrecognized digest, an entity should set a delay response timer
(0, *T*\ :sub:`delay_response`\ ] ms
(**What is the value of** *T*\ :sub:`delay_response`\ **? 200?**).
If the missing state updates are received during this time and updated root digest matches the
pending incoming interest, the delay response timer will be canceled.
If no missing state update is received during this time,
the entity should try to help the sync-interest sender to recover the knowledge set by returning
its complete knowledge set (or ignore the interest if its knowledge set is empty).


Simultaneous Update
-------------------

In some cases, two sessions may generate updates at the same time.
Some entities in the sync group may receive the update from one session,
while the rest entities may receive the update from the other session.
As a result, entities in the same sync group have different knowledge sets.
This case is equivalent to the unknown digest discussed above
and can be recovered with a complete knowledge set.
Note that ``Exclude`` selector is not used in this case.

Reset Sync Group
----------------

Sync group needs to be reset periodically in order to clean up inactive sessions.
All active sessions should join the sync group again after resetting.

Reset signal is expressed as a **reset-interest**.

    /ndn/broadcast/[AppName]/[GroupName]/reset

Reset-interest is expected to time out. Reset-interest should be sent out in two cases:

1. When a new session joins the sync group, the entity running the new session should
   send out a reset-interest.
2. Each entity, once receiving a reset-interest, should restart a timer
   (*T*\ :sub:`reset`\ , *T*\ :sub:`reset`\ + *T*\ :sub:`random`\] seconds
   (**What is the value of** *T*\ :sub:`reset`\ **? 600?**
   **What is the value of** *T*\ :sub:`random`\ **? 60?**),
   the entity should send out a reset-interest when the timer expires.

Reset-interest will multicast to all entities in the sync group.
Once receiving a reset interest, an entity should

1. reset its own sync tree but still keep using its latest sequence number;
2. express a sync-interest ``/ndn/broadcast/[AppName]/[GroupName]/[EmptyInputDigest]`` (``mustBeFresh`` =true);
3. start a delay response timer (0, *T*\ :sub:`delay_response`\ ] ms.

When the a state update is received before the timer expires,
the entity should update its sync tree and express a sync-interest with the new root digest.
The entity should reset the delay response timer until it makes the first sync-reply which contains
its own sequence number.

The ``InterestLifetime`` of reset-interest is set to 10 seconds.
If the interval between two reset-interests is less than 10 seconds, they will be aggregated.
Otherwise, the latest reset-interest will suppress the previous reset-interest.

Sync Reply Format
=================

The content of sync reply is defined as a TLV block.
::

   SyncReply ::= SYNC-REPLY-TYPE TLV-LENGTH
                 StateLeaf+

   StateLeaf ::= STATE-LEAF-TYPE TLV-LENGTH
                 Name
                 Seq

   Seq       ::= SEQ-TYPE TLV-LENGTH
                 nonNegativeInteger


Type Code Assignment
--------------------

+------------------+----------------+----------------------+
| Type             | Assigned code  | Assigned code (hex)  |
+==================+================+======================+
| SYNC-REPLY-TYPE  |  128           |  0x80                |
+------------------+----------------+----------------------+
| STATE-LEAF-TYPE  |  129           |  0x81                |
+------------------+----------------+----------------------+
| SEQ-TYPE         |  130           |  0x82                |
+------------------+----------------+----------------------+
