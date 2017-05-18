###############################
Proposal: Asynchronous PVRecord
###############################

********************
PVRecord processing
********************

==========
Background
==========

A V3 IOC has a set of DBRecords.
Each DBRecord has record support and device support attached to it.
The code that calls record support allows a record to be processed either synchronously or asynchronously.
A combination of the record and device support decides if processing is synchronous or asynchronous.


===================================
Introduction to PVRecord processing
===================================

Currently the only process method for PVRecord is: ::

    void process();

This method does not return until it is done,
i. e. it is a synchronous method.

What process does is determined by the support code attached to the record instance.
It can be an action that completes quickly or may take a long time to complete.

This proposal specifies additional process methods that support asynchronous record processing.
It is up to the support code to decide if it should be a synchronous or asynchronous record.

======================
Synchronous Processing
======================

The following makes a record synchronous. ::

    boolean isAsynRecord() { return false;}
    void process();

=======================
Asynchronous Processing
=======================

The following method makes a record asynchronous. ::

    boolean isAsynRecord() { return true;}


--------------
ChannelProcess
--------------

.. index:: Channel;process channelProcess

The following are the methods for channelProcess: ::

    void process(
       ChannelProcess channelProcess,
       ChannelProcessRequester channelProcessRequester,
       boolean block);

where

channelProcess
    The ChannelProcess that is issuing the process request.

channelProcessRequester
    The ChannelProcessRequester that called channelProcess.

block
    *true* means to call the requester only after process completes.

    *false* means to call the requester immediately.


----------
ChannelGet
----------

.. index:: Channel;process channelGet

The following are the methods for channelGet: ::

    void process(
       ChannelGet channelGet,
       ChannelGetRequester channelGetRequester,
       boolean block,
       PVCopy pvCopy,
       PVStructure pvData,
       BitSet bitSet);

where

channelGet
    The ChannelGet that is issuing the process request.

channelGetRequester
    The ChannelGetRequester that called channelGet.

block
    *true* means to call the requester only after process completes.

    *false* means to call the requester immediately.

pvCopy
    The PVCopy instance for the client.

pvData
    The client PVStructure

bitSet
    The BitSet that shows any data changes made to pvData during record processing.


----------
ChannelPut
----------

.. index:: Channel;process channelPut

The following are the methods for channelPut: ::

    void process(
       ChannelPut channelPut,
       ChannelPutRequester channelPutRequester,
       boolean block,
       PVCopy pvCopy,
       PVStructure pvData,
       BitSet bitSet);

where

channelPut
    The ChannelPut that is issuing the process request.

channelPutRequester
    The ChannelPutRequester that called channelPut.

block
    *true* means to call the requester only after process completes.

    *false* means to call the requester immediately.

pvCopy
    The PVCopy instance for the client.

pvData
    The client PVStructure

bitSet
    The BitSet that shows any data changes made to pvData during record processing.





-------------
ChannelPutGet
-------------

.. index:: Channel;process channelPutGet

The following are the methods for channelPutGet: ::

    void process(
       ChannelPutGet channelPutGet,
       ChannelPutGetRequester channelPutGetRequester,
       boolean block,
       PVCopy pvPutCopy,
       PVStructure pvPutStructure,
       BitSet putBitSet,
       PVCopy pvGetCopy,
       PVStructure pvGetStructure,
       BitSet getBitSet);
       

where

channelPutGet
    The ChannelPutGet that is issuing the process request.

channelPutGetRequester
    The ChannelPutGetRequester that called channelPutGet.

block
    *true* means to call the requester only after process completes.

    *false* means to call the requester immediately.

pvPutCopy
    The PVCopy instance for the pvPutStructure.

pvPutStructure
    The data the client sent.

putBitSet
    The BitSet that shows data changes the client made to pvPutStructure.

pvGetCopy
    The PVCopy instance for the pvGetStructure.

pvGetStructure
    The data returned to the client.

getBitSet
    The BitSet that shows any data changes made to pvGetStructure during record processing.

======================
PVCopy: record options
======================

-----
block
-----

This is used for a channel request that results in the provider issung a 
process request.
It specifies if a request should block until processing completes .


For example: ::

    pvput -r "record[block=true]field(value)" someAsynChannel Acquire

This option is honored by the following:

pvDatabase
    For channelProcess, channelGet, channelPut, and channelPutGet

pcaSrv
    For channelProcess, channelGet, and channelPut


=====================
Example: Busy Record
=====================


The following example allows a client to set the record busy and wait until the record is done.

For example: ::

    pvput -r "record[block=true]field(value)" -w 100000000 PVRBusy Acquire

Another client can set it done: ::

    pvput PVRBusy Done

The current code only allows one client at a time to set the record busy and wait for completion.


See <https://github.com/mrkraimer/exampleJava/blob/master/database/src/org/epics/exampleJava/exampleDatabase/ExampleBusyRecord.java>

==========================
Implementation of Proposal
==========================

Java code for the proposed implementation is available at:
<https://github.com/mrkraimer/pvDatabaseJava/tree/asynRecord>

The example busy record is available at:
<https://github.com/mrkraimer/exampleJava/blob/master/database/src/org/epics/exampleJava/exampleDatabase/ExampleBusyRecord.java>
</p>
