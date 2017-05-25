###############################
Proposal: Asynchronous PVRecord
###############################

**Author:** Marty Kraimer

**Date:** 2017.05.25

****************
Proposal Summary
****************

Currently the only process method for PVRecord is: ::

    void process();

This method does not return until it is done. Thus it is a synchronous method.

This proposal adds the following methods: ::

    boolean isAsynRecord();
    void process(                         // For ChannelProcess
       ChannelProcess channelProcess,
       ChannelProcessRequester channelProcessRequester,
       boolean block);
    void PVRecord::process(               // For ChannelGet
       ChannelGet channelGet,
       ChannelGetRequester channelGetRequester,
       boolean block,
       PVCopy pvCopy,
       PVStructure pvData,
       BitSet bitSet);
    void process(                         // For ChannelPut
       ChannelPut channelPut,
       ChannelPutRequester channelPutRequester,
       boolean block,
       PVCopy pvCopy,
       PVStructure pvData,
       BitSet bitSet);
    void process(                         // For ChannelPutGet
       ChannelPutGet channelPutGet,
       ChannelPutGetRequester channelPutGetRequester,
       boolean block,
       PVCopy pvPutCopy,
       PVStructure pvPutStructure,
       BitSet putBitSet,
       PVCopy pvGetCopy,
       PVStructure pvGetStructure,
       BitSet getBitSet);
    
PVRecord is a base class.
It implements **isAsynRecord** by always returning false.
It implements the other new methods by always calling the requester with a status that indicates
failure.

Code can extend PVRecord and implement any of the above methods desired.

The rest of this documemt provides material that attempts to explain the proposal in more detail.


*******************
EPICS V4 Background
*******************

This is a brief summary EPICS V4.
It only discusses the EPICS V4 components required to understand this proposal.

A more complete introduction to EPICS V4 is provided in:

<http://epics-pvdata.sourceforge.net/informative/developerGuide/developerGuide.html>

This background is intended for readers that have at least some knowlege of object oriented languages. In particular Java or C++. The syntax in this proposal is
psuedo code similar to Java but simplified. 

EPICS V4 has a number of modules.
Most are implemented in both Java and C++.
In order to understand this proposal, some knowledge of the modules
**pvData**, **pvAccess**, and **pvDatabase** is required.
The following discussion applies to both the Java and C++ implementation of each module.



**pvData** provides a way to define and access structured data.


**pvAccess** provides communication between a client and a server. This includes network suport.
All data is passed as **pvData** structures.

**pvDatabase** provides a way to create PVRecords. A PVRecord holds a memory resident **pvData**
structure. In addition **pvDatabase** provides a **pvAccess** provider that allows a client to access the PVRecords.


======
pvData
======

The following are the **pvData** components of interest to this proposal.

-------------------
pvData: PVStructure
-------------------

A **PVStructure** instance holds a set of structured data.

The following code: ::

    FieldCreate fieldCreate = FieldFactory.getFieldCreate();
    FieldBuilder fb = fieldCreate.createFieldBuilder();
    StandardField standardField = StandardFieldFactory.getStandardField();
    Structure structure =
            fb.add("value", ScalarType.pvDouble).
            add("alarm",standardField.alarm()).
            add("timeStamp",standardField.timeStamp()).
            createStructure();
    PVStructure pvStructure = pvDataCreate.createPVStructure(structure);
    System.out.println(pvStructure);

produces: ::

    structure 
        double value 0
        alarm_t alarm
            int severity 0
            int status 0
            string message 
        time_t timeStamp
            long secondsPastEpoch 0
            int nanoseconds 0
            int userTag 0

In this proposal many of the arguments are a PVStructure.


For example: ::

    void someMethod(
       ...
       PVStructure pvData,    // PVStructure is an instance of a structure.
       ...);

--------------
pvData: BitSet
--------------

**pvData** provides an API and implementation for a bitSet that can be associated with
a top level PVStructure. 
The bitSet assigns a bit to each field of the PVStructure.


--------------
pvData: PVCopy
--------------


**pvData** also provides an API and implementation for a facility named **pvCopy**.
An instance of **pvCopy** accesses a subset of the fields in a top level PVStructure
and also allows options to be associated with each field in the subset.

========
pvAccess
========

**pvAccess** provides both client and server support for passing **pvData** objects between client and server.


Any data source that holds data as **pvData** objects can also implement a **ChannelProvider**,
and register the provider with **pvAccess**. 
It is then a server.
A provider can provide access to an arbitrary number of objects but
each object must have a unique channelName and a top level **PVStructure**.

A client connects to a provider by asking to connect to a channelName.
The provider returns a **Channel** object to the client.

Of interest to this proposal are the following **Channel** methods:

**createChannelProcess**

    A **ChannelProcess** provides a method that allows the client to ask that the server process.


**createChannelGet**

    A **ChannelGet** provides methods that allow the client to get a subset of the data in
    the top level **PVStructure** for the channel.


**createChannelPut**

    A **ChannelPut** provides methods that allow the client to put to a subset of the data in
    the top level **PVStructure** for the channel.


**createChannelPutGet**

    A **ChannelPutGet** provides methods that allow the client to put to a subset of the data in
    the top level **PVStructure** for the channel. Then the record is normally processed.
    Finally a different subset of the data in
    the top level **PVStructure** is returned to the client.


The **pvAccess** API is callback based.
Thus associated with many client requests is a client supplied callback.
As a simple example consider **ChannelProcess**. ::

    class Channel {
        ...
        ChannelProcess createChannelProcess(
            ChannelProcessRequester channelProcessRequester,
    		PVStructure pvRequest);
        ...
     }
     class ChannelProcessRequester{
         void channelProcessConnect(Status status, ChannelProcess channelProcess);
         void processDone(Status status, ChannelProcess channelProcess);
     }
     class ChannelProcess {
         void process();
     }

When the client calls **ChannelProcess::process**, the method issues the request and returns immediately.
When the process request completes **ChannelProcessRequester::processDone** is called.

The same pattern occurs for **ChannelGet**, **ChannelPut**, and **ChannelPutGet**.




==========
pvDatabase
==========

**pvDatabase** is one of the EPICS V4 modules.

It provides two main features:

**PVDatabase**

    This is a memory resident database of **PVRecord** instances.
    Each record has data composed a top level **PVStructure**.
    A base class that is a complete implemention of **PVRecord** is provided.
    Other code can be provided that extends the base class.

**ChannelProviderLocal**

    This is a channel provider for accessing **PVRecord** instances.
    Of interest to this proposal is that it implements **ChannelProcess**,
    **ChannelGet**, **ChannelPut**, and **ChannelPutGet**.


===================================
PVRecord processing
===================================

Currently the only process method for PVRecord is: ::

    void PVRecord::process();

This method does not return until it is done,
i. e. it is a synchronous method.

What process does is determined by the support code attached to the record instance.
It can be an action that completes quickly or may take a long time to complete.

This method is called by **ChannelProviderLocal**.
It is called by the implementation of **ChannelProcsss**, **ChannelGet**, **ChannelPut**, and **ChannelPutGet**.

This proposal specifies additional process methods that support asynchronous record processing.
It is up to the support code to decide if it should be a synchronous or asynchronous record.

======================
Synchronous Processing
======================

The following makes a record synchronous. ::

    boolean PVRecord::isAsynRecord() { return false;}
    void process();

=======================
Asynchronous Processing
=======================

The following method makes a record asynchronous. ::

    boolean PVRecord::isAsynRecord() { return true;}



--------------
ChannelProcess
--------------

.. index:: Channel;process channelProcess

The following is the method for channelProcess: ::

    void PVRecord::process(
       ChannelProcess channelProcess,
       ChannelProcessRequester channelProcessRequester,
       boolean block);

where

channelProcess
    The ChannelProcess instance that is issuing the process request.

channelProcessRequester
    The ChannelProcessRequester instance that called channelProcess.

block
    *true* means to call the requester only after process completes.

    *false* means to call the requester immediately.

The implementation of **ChannelProviderLocal::ChannelProcess** calls this as follows: ::

    void process() {
        if(pvRecord.isAsynRecord()) {
            pvRecord.process(this,channelProcessRequester,block);
            return;
        }
        pvRecord.lock();
        try {
            pvRecord.beginGroupPut();
            pvRecord.process();
            pvRecord.endGroupPut();
        } finally {
            pvRecord.unlock();
        }
        channelProcessRequester.processDone(okStatus,this);
    }


----------
ChannelGet
----------

.. index:: Channel;process channelGet

The following is the method for channelGet: ::

    void PVRecord::process(
       ChannelGet channelGet,
       ChannelGetRequester channelGetRequester,
       boolean block,
       PVCopy pvCopy,
       PVStructure pvData,
       BitSet bitSet);

where

channelGet
    The ChannelGet instance that is issuing the process request.

channelGetRequester
    The ChannelGetRequester instance that called channelGet.

block
    *true* means to call the requester only after process completes.

    *false* means to call the requester immediately.

pvCopy
    The PVCopy instance for the client.

pvData
    The client PVStructure.

bitSet
    The BitSet that shows any data changes made to pvData since the last get request..

The implementation of **ChannelProviderLocal::ChannelGet** calls this as follows: ::

    void get() {
        if(pvRecord.isAsynRecord() && callProcess) {
            pvRecord.process(this,channelGetRequester,block,pvCopy,pvStructure,bitSet);
            return;
        }
        bitSet.clear();
        pvRecord.lock();
        try {
            if(callProcess) {
                pvRecord.beginGroupPut();
                pvRecord.process();
                pvRecord.endGroupPut();
            }
            pvCopy.updateCopySetBitSet(pvStructure, bitSet);
        } finally {
            pvRecord.unlock();
        }
        if(firstTime) {
            bitSet.clear();
            bitSet.set(0);
            firstTime = false;
        }
        channelGetRequester.getDone(okStatus,this,pvStructure,bitSet);
    }



----------
ChannelPut
----------

.. index:: Channel;process channelPut

The following is the method for channelPut: ::

    void PVRecord::process(
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
    The BitSet that shows changes the client made to pvData.


The implementation of **ChannelProviderLocal::ChannelPut** calls this as follows: ::

    void put(PVStructure pvPutStructure, BitSet bitSet) {
        if(isDestroyed) {
            channelPutRequester.putDone(requestDestroyedStatus,this);
            return;
        }
        if(pvRecord.isAsynRecord() && callProcess) {
            pvRecord.process(this,channelPutRequester,block,pvCopy,pvPutStructure,bitSet);
            return;
        }
        pvRecord.lock();
        try {
            pvRecord.beginGroupPut();
            pvCopy.updateMaster(pvPutStructure,bitSet);
            if(callProcess) {
                pvRecord.process();
            }
            pvRecord.endGroupPut();
        } finally {
            pvRecord.unlock();
        }
        if(pvRecord.getTraceLevel()>1) {
            System.out.println("ChannelPutLocal::put recordName " + pvRecord.getRecordName());
        }
        channelPutRequester.putDone(okStatus,this);
    }


-------------
ChannelPutGet
-------------

.. index:: Channel;process channelPutGet

The following is the method for channelPutGet: ::

    void PVRecord::process(
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
    The BitSet that shows any data changes made to pvGetStructure since the last putGet request.

The implementation of **ChannelProviderLocal::ChannelPutGet** calls this as follows: ::

    void putGet(PVStructure pvPutStructure, BitSet putBitSet)
    {
        if(isDestroyed) {
            channelPutGetRequester.putGetDone(requestDestroyedStatus,this,null,null);
            return;
        }
        if(pvRecord.isAsynRecord() && callProcess) {
            pvRecord.process(this,channelPutGetRequester,block,
                    pvPutCopy,pvPutStructure,putBitSet,
                    pvGetCopy,pvGetStructure,getBitSet);
            return;
        }
        pvRecord.lock();
        try {
            pvRecord.beginGroupPut();
            pvPutCopy.updateMaster(pvPutStructure, putBitSet);
            if(callProcess) pvRecord.process();
            getBitSet.clear();
            pvGetCopy.updateCopySetBitSet(pvGetStructure, getBitSet);
            pvRecord.endGroupPut();
        } finally {
            pvRecord.unlock();
        }
        if(pvRecord.getTraceLevel()>1) {
            System.out.println("ChannelPutGetLocal::putGet recordName " + pvRecord.getRecordName());
        }
        channelPutGetRequester.putGetDone(okStatus,this,pvGetStructure,getBitSet);
    }

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
    For channelProcess, channelGet, channelPut, and channelPutGet.
    **Note:** Only if this proposal is accepted and implemented.
 
pvaSrv
    For channelProcess, channelGet, and channelPut


=====================
Example: Busy Record
=====================

The example busy record is available at:

<https://github.com/mrkraimer/exampleJava/blob/master/database/src/org/epics/exampleJava/exampleDatabase/ExampleBusyRecord.java>

The example allows a client to set the record busy and wait until the record is done.

For example: ::

    pvput -r "record[block=true]field(value)" -w 100000000 PVRBusy Acquire

Another client can set it done: ::

    pvput PVRBusy Done

**Notes**

1) The example only allows one client at a time to set the record busy and wait for completion. It should be changed to allow multiple clients to wait for completion.

2) The example should also implement the methods for channelProcess, channelGet, and channelPutGet


==========================
Implementation of Proposal
==========================

Java code for the proposed implementation is available at:

<https://github.com/mrkraimer/pvDatabaseJava/tree/asynRecord>


**Note** If proposal is accepted the changes must undergo more testing and must also be implemented in **pvDatabaseCPP**.



