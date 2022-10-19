Dqlite (distributed SQLite) is a fast, embedded and persistent SQL database that is designed to keep your application running.

It extends [SQLite](https://www.sqlite.org/) across a cluster of machines, which ensures high availability and fast failover with automatic leader election.
To minimise transaction latency and gain high-performance transactional consensus and fault tolerance, it uses C-Raft, an optimised Raft implementation in C.

Dqlite provides edge applications with an ultra-fast SQL database that is automatically replicated across multiple hosts and guarantees fast failover.
It ensures maximum cross-platform portability, outstanding efficiency and a tiny footprint.

These features make Dqlite perfect for fault-tolerant IoT and Edge devices, but also for agents and backend cloud services that want to simplify their operation.

At the moment, the biggest user of Dqlite is the [LXD system containers manager](https://www.linuxcontainers.org), which uses Dqlite to implement high-availability when running in cluster mode.

## Project and community

Dqlite is a Canonical project. It is released under a slightly modified version of LGPLv3, which includes a copyright exception allowing users to statically link the library code in their project and release the final work under their own terms.

Dqlite is an open source project that warmly welcomes community projects, contributions, suggestions, fixes and constructive feedback.

- [License](https://github.com/canonical/dqlite/blob/master/LICENSE) <!-- wokeignore:rule=master -->
- [Get support through Ubuntu Pro](https://ubuntu.com/support)
- [Join the Discourse forum to ask questions](https://discourse.dqlite.io/)
- [Report bugs](https://github.com/canonical/dqlite/issues)

Thinking about using Dqlite for your next project? [Get in touch](https://canonical.com/contact-us)!

## Navigation

[details=Navigation]
| Level | Path | Navlink |
| -- | -- | -- |
| 1 | | [Dqlite documentation](/t/dqlite-documentation/34) |
| 0 | | How-to guides |
| 1 | howto/get-started | [Get started](tbd) |
| 0 | | Explanation |
| 1 | explanation/architecture | [Architecture](/t/architecture/27) |
| 1 | explanation/consistency-model | [Consistency Model](/t/consistency-model/29) |
| 1 | explanation/replication | [Replication](/t/replication/28) |
| 1 | explanation/faq | [FAQ](/t/documentation-faq/22) |
| 0 | | Reference |
| 1 | reference/wire-protocol | [Wire Protocol](/t/wire-protocol/23) |
[/details]

## Redirects

[details=Mapping table]
| Path               | Location                       |
| ----               | --------                       |
| /architecture      | /explanation/architecture      |
| /consistency-model | /explanation/consistency-model |
| /replication       | /explanation/replication       |
| /faq               | /explanation/faq               |
| /protocol          | /reference/wire-protocol       |
[/details]
