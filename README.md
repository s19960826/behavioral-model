# BEHAVIORAL MODEL REPOSITORY

[![Build Status](https://travis-ci.org/p4lang/behavioral-model.svg?branch=master)](https://travis-ci.org/p4lang/behavioral-model)

This is the second version of the P4 software switch (aka behavioral model),
nicknamed bmv2. It is meant to replace the original version, p4c-behavioral, in
the long run, although we do not have feature equivalence yet. Unlike
p4c-behavioral, this new version is static (i.e. we do not need to auto-generate
new code and recompile every time a modification is done to the P4 program) and
written in C++11. For information on why we decided to write a new version of
the behavioral model, please look at the FAQ below.

## Dependencies

On Ubuntu 14.04, the following packages are required:

- automake
- libjudy-dev
- libgmp-dev
- libpcap-dev
- libboost-dev
- libboost-test-dev
- libboost-program-options-dev
- libboost-system-dev
- libboost-filesystem-dev
- libboost-thread-dev
- libevent-dev
- libtool
- flex 
- bison
- pkg-config
- g++
- libssl-dev

You also need to install *thrift* and *nanomsg* from source. Feel free to use
the install scripts under build/travis/.

Our Travis regression tests run on Ubuntu 12.04. Look at .travis.yml for more
information on the Ubuntu 12.04 dependencies.

## Building the code

    1. ./autogen.sh
    2. ./configure
    3. make

To enable logging, you probably want to use the following flags when running
configure:

    ./configure 'CPPFLAGS=-DELOGGER_NANOMSG -DENABLE_SIMPLELOG'

In 'debug mode', you probably want to also use the following as well:

    'CXXFLAGS=-O0 -g'

## Running the tests

To run the unit tests, simply do:

    make check

**If you get a nanomsg error when running the tests (make check), try running
  them as sudo**

## Running your P4 program

To run your own P4 programs in bmv2, you first need to transform the P4 code
into a json representation which can be consummed by the software switch. This
representation will tell bmv2 which tables to initialize, how to cinfigure the
parser, ... It is produced by the [p4c-bm](https://github.com/p4lang/p4c-bm)
tool. Please take a look at the
[README](https://github.com/p4lang/p4c-bm/blob/master/README.rst) for this repo
to find out how to install it. Once this is done, you can obtain the json file
as follows:

    p4c-bm --json <path to JSON file> <path to P4 file>

The json file can now be 'fed' to bmv2. Assuming you are using the
*simple_switch* target:

    sudo ./simple_switch -i <iface0> -i <iface1> <path to JSON file>

In this example \<iface0\> and \<iface1\> are the interfaces which are bound to
the switch (as ports 0 and 1).

## Using the CLI to populate tables...

The CLI code can be found at [tools/runtime_CLI.py](tools/runtime_CLI.py). It
also requires the json file generated by p4c-bm and can be used like this:

    ./runtime_CLI.py --json <path to JSON file> --thrift-port 9090

The CLI connect to the Thrift RPC server running in each switch process. 9090 is
the default value but of course if you are running several devices on your
machine, you will need to provide a different port for each. One CLI instance
can only connect to one switch device.

The CLI is realized using the Python's cmd module and supports
auto-completion. If you inspect the code, you will see that the list of
supported commands. This list includes:

    - table_set_default <table name> <action name> <action parameters>
    - table_add <table name> <action name> <match fields> => <action parameters> [priority]
    - table_delete <table name> <entry handle>

The CLI include commands to program the default multicast engine. Unfortunately
they will not work with the *simple_switch* target which uses a more advanced
engine (support will be added soon). They will work with the *l2_switch* target
though.

You can take a look at the *commands.txt* file for
[*l2_switch*](targets/l2_switch/commands.txt) and 
[*simple_router*](targets/simple_router/commands.txt) to see how the CLI can be
used.

## Displaying the event logging messages

Use [tools/nanomsg_client.py](tools/nanomsg_client.py) as follows when the
switch is running:

    sudo ./nanomsg_client.py <path to JSON file>

The script will display events of significance (table hits / misses, parser
transitions, ...) for each packet.

## Integrating with Mininet

We will provide more information in a separate document. However you can test
the Mininet integration right away using our *simple_router* target.

In a first terminal, type the following:

    - cd mininet
    - sudo python 1sw_demo.py --behavioral-exe ../targets/simple_router/simple_router --json ../targets/simple_router/simple_router.json

Then in a second terminal:

    - cd targets/simple_router
    - ./runtime_CLI --json simple_router.json < commands.txt

Now the switch is running and the tables have been populated. You can run
*pingall* in Mininet or start a TCP flow with iperf between hosts *h1* and *h2*.

## FAQ

### Why did we need bmv2 ?

- the new C++ code is not auto-generated for each P4 program. This means that it
  becomes very easy and very fast to change your P4 program and test it
  again. The whole P4 development process becomes more efficient. Every time you
  change your P4 program, you simply need to produce the json for it using
  p4c-bm and feed it to the bmv2 executable.
- because the bmv2 code is not auto-generated, we hope it is easier to
  understand. We hope this will encourage the community to contribute even more
  to the P4 software switch.
- using the auto-generated PD library (which of course still needs to be
  recompiled for each P4 program) is now optional. We provide an intuitive CLI
  which can be used to program the runtime behavior of each switch device.
- the new code is target independent. While the original p4c-behavioral assumed
  a fixed abstract switch model with 2 pipelines (ingress and egress), bmv2
  makes no such assumption and can be used to represent many switch
  architectures. Three different -although similar- such architectures can be
  found in the targets/ directory. If you are a networking company interested in
  programming your device (parser, macth-action pipeline, deparser) with P4, you
  can use bmv2 to reproduce the behavior of your device.

### How do program my own target / switch architecture using bmv2 ?

You can take a look at the targets/ directory. We will soon publish a separate
document with detailed information.

### What else is new in bmv2 ?

- arithmetic is now possible on arbitrarily wide fields (no more limited to <=
  32-bit fields).
- we finally have unit tests!
- while it is still incomplete, we provide a convenient 'event-logger' built on
  top of nanomsg. Every time a 'significant' event happens (e.g. table hit,
  parser transition,...) a message is broadcast on a nanomsg channel and any
  client can consume it.

### How can I contribute ?

You can fork the repo and submit a pull request in Github. For more information
send us an email (antonin@barefootnetworks.com).
