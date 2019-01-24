
SNMP Object Mapper
------------------

[![PyPI](https://img.shields.io/pypi/v/snmpobjects.svg?maxAge=2592000)](https://pypi.org/project/snmpobjects)
[![Python Versions](https://img.shields.io/pypi/pyversions/snmpobjects.svg)](https://pypi.org/project/snmpobjects/)
[![Status](https://img.shields.io/pypi/status/snmpobjects.svg)](https://github.com/etingof/snmpobjects/)
[![Build status](https://travis-ci.org/etingof/snmpobjects.svg?branch=master)](https://travis-ci.org/etingof/snmpobjects)
[![GitHub license](https://img.shields.io/badge/license-BSD-blue.svg)](https://raw.githubusercontent.com/etingof/snmpobjects/master/LICENSE.txt)

**WARNING**: *what's following is NOT yet supported by any code. These are the ideas based
on the inspiring conversation with `Jack Wanke` to further contemplate on. This is not
the usage guide (yet!).*

The SNMP objects library aims at representing remote SNMP table as a collection
of mutable Python objects. In a sense it can be viewed as an
object-relational-mapper with the difference that SQL database or a typical
ORM is replaced with a remotely accessible SNMP agent.

Perhaps one of the goals behind the SNMP objects package is to piggyback
the existing awareness of the software engineers of Python ORMs and built-in
Python types to represent otherwise quite foreign SNMP data model in a more
conventional form.

The structure and other details of SNMP conceptual tables are defined in
the SNMP Management Information Base files (MIB). SNMP objects library
relies on the [pysmi](https://snmplabs.com/pysmi) project to compile SNMP
MIB definitions into Python classes built on top of the *snmpobjects* data
model.

The actual data is read from and possibly committed to the remote SNMP
agent(s) accessible through SNMP v1/v2c/v3. These SNMP interactions are
mostly hidden from the user of the *snmpobjects* library.

Features
--------

* Familiar ORM-like programming interface to SNMP tables
* Backend SNMP agent accessible through SNMPv1/v2c/v3
* Supports asynchronous MIB objects API
* Extension modules supporting local caching and and on-the-fly data modification
* Works on Linux, Windows and OS X

SNMP objects showcase
---------------------

Let's take [IF-MIB](http://mibs.snmplabs.com/asn1/IF-MIB)*::ifTable* as an example:

```bash
ifEntry OBJECT-TYPE
    SYNTAX      IfEntry
    MAX-ACCESS  not-accessible
    STATUS      current
    DESCRIPTION
            "An entry containing management information applicable to a
            particular interface."
    INDEX   { ifIndex }
    ::= { ifTable 1 }

IfEntry ::=
    SEQUENCE {
        ifIndex                 InterfaceIndex,
        ifDescr                 DisplayString,
        ifType                  IANAifType,
        ifMtu                   Integer32,
        ifSpeed                 Gauge32,
        ifPhysAddress           PhysAddress,
        ifAdminStatus           INTEGER,
        ifOperStatus            INTEGER,
        ifLastChange            TimeTicks,
        ifInOctets              Counter32,
        ifInUcastPkts           Counter32,
        ifInNUcastPkts          Counter32,  -- deprecated
        ifInDiscards            Counter32,
        ifInErrors              Counter32,
        ifInUnknownProtos       Counter32,
        ifOutOctets             Counter32,
        ifOutUcastPkts          Counter32,
        ifOutNUcastPkts         Counter32,  -- deprecated
        ifOutDiscards           Counter32,
        ifOutErrors             Counter32,
        ifOutQLen               Gauge32,    -- deprecated
        ifSpecific              OBJECT IDENTIFIER -- deprecated
    }
```

Through a *pysmi* template, this MIB definition could be automatically turned into
*snmpobjects* class like this:

```python

class IfTable(SnmpTable):
    """Represents SNMP table `IF-MIB::ifTable`
    
    A list of interface entries.  The number of entries is
    given by the value of ifNumber.
    """
    ifIndex = InterfaceIndex
    ifDescr = DisplayString
    ifType = IANAifType
    ifMtu = Integer32
    ifSpeed = Gauge32
    ifPhysAddress = PhysAddress
    ifAdminStatus = Integer
    ifOperStatus = Integer
    ifLastChange = TimeTicks
    ifInOctets = Counter32
    ifInUcastPkts = Counter32
    ifInNUcastPkts = Counter32
    ifInDiscards = Counter32
    ifInErrors = Counter32
    ifInUnknownProtos = Counter32
    ifOutOctets = Counter32
    ifOutUcastPkts = Counter32
    ifOutNUcastPkts = Counter32
    ifOutDiscards = Counter32
    ifOutErrors = Counter32
    ifOutQLen = Gauge32
    ifSpecific = ObjectIdentifier
```

Since the actual data is held at some remote SNMP agent, we need to bind SNMP
table object model to some particular SNMP agent:

```python
agent = agent_factory(
    transport_address=('127.0.0.1', 161),
)

ifTable = IfTable.backed_by(agent)

```

With *snmpobjects* model, SNMP table object mimics Python ordered `dict`.

The entries of SNMP tables are always addressed by one or more indices. For example,
to access the first interface in the table above one would have to use index *1*.

The indices are treated as keys to the SNMP tables objects.

```python
ifTableRow = ifTable[IfTable.indices(ifIndex=1)]

print(ifTableRow.ifDescr)
```

With *snmpobjects* data mode, the table row instance behaves like a `namedtuple`
with names taken from the SNMP table MIB definition.

Sometimes the exact index may not be known beforehand e.g. in case of dynamic SNMP
tables. So we could just iterate the table:

```python
for ifTableRow in ifTable:
    print(ifTableRow)
```

SNMP indices could be compound in the sense that each entry is addressed by a joint
set of indices some of which may be known in advance. In that case we could iterate
through the subset of the rows by slicing the sequence by known indices:

```python
for ifTableRow in ifTable[IfTable.indices(ifIndex=3):]:
    print(ifTableRow)
```

Some columns of SNMP tables can be writable. With *snmpobjects* column write looks
like this:

```python
ifTable[IfTable.indices(ifIndex=1)] = {'ifAdminStatus': 'up'}
ifTable.save()
```

You just need to assign any of:

* a Python mapping having the keys of SNMP table columns that needs to be set or
* a Python object exposing column names as its attributes or
* a Python sequence listing the columns to be set starting from the first column

SNMP tables offer the mechanism of the entire row creation/removal through the
dedicated `RowStatus` column. If `ifTable` was designed writable, it would have have
one. Then row creation operation would be simply `status` column assignment:

```python
ifTable[IfTable.indices(ifIndex=1)] = {'status': 'createAndWait'}
ifTable.save()
```

The final `.save()` call maps on SNMP SET command. SNMP GET, GETNEXT and GETBULK
command are issued lazily whenever data read deemed necessary.

Download & Install
------------------

When the SNMP objects software becomes available, it could be freely downloaded from
[PyPI](https://pypi.org/project/snmpobjects).

Just run:

```bash
$ pip install snmpobjects
```

Alternatively, you will be able to get it from [GitHub](https://github.com/etingof/snmpobjects/releases).

Getting help
------------

If something will not work as expected or we are missing an interesting feature,
[open an issue](https://github.com/etingof/snmpobjects/issues) at GitHub or
post your question [on Stack Overflow](https://stackoverflow.com/questions/ask).

Finally, your PRs are warmly welcome! ;-)

Copyright (c) 2019, [Ilya Etingof](mailto:etingof@gmail.com). All rights reserved.
