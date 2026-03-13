# Suricata Turbo: Hardware Flow Offload for Suricata IDS

## Overview

Suricata Turbo adds hardware traffic filtering offload to Suricata, the leading open-source intrusion detection engine (IDS) used by many private and public organizations. When an IDS can't keep up with traffic, it drops packets at random, and threats pass uninspected. Suricata already has a software optimization to drop uninteresting traffic at the start of the processing (e.g., encrypted traffic or elephant flows), but the CPU still handles every packet, even when the decision is to ignore it. Suricata Turbo adds a new offload mechanism and pushes filtering of the benign traffic into the network card itself. Traffic that doesn't need inspection never reaches the CPU, freeing processing power for connections that actually warrant deep inspection. Suricata still tracks offloaded flows through the hardware connection table, so nothing disappears from its view.

Hardware offload supports both preconfigured filtering rules (e.g., to avoid daily inspection of massive volumes of benign backup traffic) and dynamic filtering driven by Suricata's own detection policies at runtime. It works on commodity network cards from multiple vendors with no proprietary software required and is built to drop into existing Suricata deployments.

## Architecture

Suricata Turbo extends Suricata's high-speed DPDK capture path with a hardware flow offload engine built on rte_flow, DPDK's vendor-neutral interface for programming flow-level filtering rules directly on the NIC.

### Static Offload

Static offload rules are defined in the Suricata YAML configuration and installed on the NIC at startup. These rules match on standard flow fields (5-tuple, VLAN, protocol type) and instruct the NIC hardware to filter matching packets before they reach any CPU queue. Typical use cases include filtering known infrastructure traffic, management VLANs, backup replication streams, or specific protocol classes that are not relevant for security inspection. The static rule engine accepts rte_flow patterns in testpmd CLI syntax, parsed by a library extracted from DPDK testpmd's own flow command parser and rebuilt as a standalone component currently being merged upstream into DPDK. Any pattern expressible in testpmd syntax is available for static offload.

### Dynamic Offload

Dynamic offload hooks into Suricata's native bypass API. When a flow worker decides a flow qualifies for hardware offload, it sends the request to a dedicated bypass thread manager through a DPDK ring buffer. The flow worker returns immediately to processing packets and is never blocked waiting for an rte_flow rule to be programmed on the NIC. The bypass manager drains the ring and applies offload requests sequentially. On successful rte_flow creation, the hardware flow handle is stored in Suricata's software flow table, linking the NIC-level rule to Suricata's internal flow tracking.

Dynamic offload matches on 5-tuples. Supported policies include post-handshake TLS/QUIC filtering, elephant flow detection by byte or packet count, and subnet allowlisting. The policy configuration in the YAML or in the ruleset determines what qualifies. Allowlists can be reloaded at runtime through Suricata's Unix socket without restarting the engine.

When the NIC hardware flow table is full, new offload requests fall back to software bypass transparently. Suricata exposes this condition through its counters, so operators can monitor how often hardware capacity is reached. NIC flow table sizes vary by hardware, ranging from roughly 65,000 to millions of concurrent rules. The tool has been validated on multiple NIC platforms from different vendors (Intel, NVIDIA, DYNANIC) with no proprietary SDK.

### NIC Group Architecture

The implementation uses a two-level rte_flow group architecture on supported NICs:

- **Group 0**: Static drop-filter rules (highest priority) and a catch-all JUMP rule that forwards remaining traffic to group 1.
- **Group 1**: Dynamic bypass rules (5-tuple match with COUNT+DROP actions) and RSS distribution.

This separation keeps the small, firmware-limited group 0 table free for static rules while allowing thousands of dynamic bypass rules in the larger group 1 table.

---

# Suricata

[![Fuzzing Status](https://oss-fuzz-build-logs.storage.googleapis.com/badges/suricata.svg)](https://bugs.chromium.org/p/oss-fuzz/issues/list?sort=-opened&can=1&q=proj:suricata)
[![codecov](https://codecov.io/gh/OISF/suricata/branch/master/graph/badge.svg?token=QRyyn2BSo1)](https://codecov.io/gh/OISF/suricata)

## Introduction

[Suricata](https://suricata.io) is a network IDS, IPS and NSM engine
developed by the [OISF](https://oisf.net) and the Suricata community.

## Resources

- [Home Page](https://suricata.io)
- [Bug Tracker](https://redmine.openinfosecfoundation.org/projects/suricata)
- [User Guide](https://docs.suricata.io)
- [Dev Guide](https://docs.suricata.io/en/latest/devguide/index.html)
- [Installation Guide](https://docs.suricata.io/en/latest/install.html)
- [User Support Forum](https://forum.suricata.io)

## Contributing

We're happily taking patches and other contributions. Please see our
[Contribution
Process](https://docs.suricata.io/en/latest/devguide/contributing/contribution-process.html)
for how to get started.

Suricata is a complex piece of software dealing with mostly untrusted
input. Mishandling this input will have serious consequences:

* in IPS mode a crash may knock a network offline
* in passive mode a compromise of the IDS may lead to loss of critical
  and confidential data
* missed detection may lead to undetected compromise of the network

In other words, we think the stakes are pretty high, especially since
in many common cases the IDS/IPS will be directly reachable by an
attacker.

For this reason, we have developed a QA process that is quite
extensive. A consequence is that contributing to Suricata can be a
somewhat lengthy process.

On a high level, the steps are:

1. GitHub-CI based checks. This runs automatically when a pull request
   is made.
2. Review by devs from the team and community
3. QA runs from private QA setups. These are private due to the nature
   of the test traffic.

### Overview of Suricata's QA steps

OISF team members are able to submit builds to our private QA
setup. It will run a series of build tests and a regression suite to
confirm no existing features break.

The final QA runs takes a few hours minimally, and generally runs
overnight. It currently runs:

- extensive build tests on different OS', compilers, optimization
  levels, configure features
- static code analysis using cppcheck, scan-build
- runtime code analysis using valgrind, AddressSanitizer,
  LeakSanitizer
- regression tests for past bugs
- output validation of logging
- unix socket testing
- pcap based fuzz testing using ASAN and LSAN
- traffic replay based IDS and IPS tests

Next to these tests, based on the type of code change further tests
can be run manually:

- traffic replay testing (multi-gigabit)
- large pcap collection processing (multi-terabytes)
- fuzz testing (might take multiple days or even weeks)
- pcap based performance testing
- live performance testing
- various other manual tests based on evaluation of the proposed
  changes

It's important to realize that almost all of the tests above are used
as acceptance tests. If something fails, it's up to you to address
this in your code.

One step of the QA is currently run post-merge. We submit builds to
the Coverity Scan program. Due to limitations of this (free) service,
we can submit once a day max.  Of course it can happen that after the
merge the community will find issues. For both cases we request you to
help address the issues as they may come up.

## FAQ

__Q: Will you accept my PR?__

A: That depends on a number of things, including the code
quality. With new features it also depends on whether the team and/or
the community think the feature is useful, how much it affects other
code and features, the risk of performance regressions, etc.

__Q: When will my PR be merged?__

A: It depends, if it's a major feature or considered a high risk
change, it will probably go into the next major version.

__Q: Why was my PR closed?__

A: As documented in the [Suricata GitHub
workflow](https://docs.suricata.io/en/latest/devguide/contributing/github-pr-workflow.html),
we expect a new pull request for every change.

Normally, the team (or community) will give feedback on a pull request
after which it is expected to be replaced by an improved PR. So look
at the comments. If you disagree with the comments we can still
discuss them in the closed PR.

If the PR was closed without comments it's likely due to QA
failure. If the GitHub-CI checks failed, the PR should be fixed right
away. No need for a discussion about it, unless you believe the QA
failure is incorrect.

__Q: The compiler/code analyser/tool is wrong, what now?__

A: To assist in the automation of the QA, we're not accepting warnings
or errors to stay. In some cases this could mean that we add a
suppression if the tool supports that (e.g. valgrind, DrMemory). Some
warnings can be disabled. In some exceptional cases the only
'solution' is to refactor the code to work around a static code
checker limitation false positive. While frustrating, we prefer this
over leaving warnings in the output. Warnings tend to get ignored and
then increase risk of hiding other warnings.

__Q: I think your QA test is wrong__

A: If you really think it is, we can discuss how to improve it. But
don't come to this conclusion too quickly, more often it's the code
that turns out to be wrong.

__Q: Do you require signing of a contributor license agreement?__

A: Yes, we do this to keep the ownership of Suricata in one hand: the
Open Information Security Foundation. See
http://suricata.io/about/open-source/ and
http://suricata.io/about/contribution-agreement/
