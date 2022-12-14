The code of the Dqlite project is available on [GitHub](https://github.com/canonical/dqlite).

If you want to see Dqlite in action, check out the [demo program](https://github.com/canonical/go-dqlite#demo) that comes with the Go Dqlite bindings.

For details about Dqlite's internals, see the [architecture](/t/architecture/27) notes, the [consistency model](/t/consistency-model/29) description and the [replication](/t/replication/28) mechanics. Alternatively, you can watch the [talk about Dqlite](https://fosdem.org/2020/schedule/event/dqlite/) that was given at FOSDEM 2020.

To start developing, check out the following information:

- If you’re writing an application, refer to the Dqlite C [header file](https://github.com/canonical/dqlite/blob/master/include/dqlite.h) <!-- wokeignore:rule=master --> or the [`go-dqlite`](https://github.com/canonical/go-dqlite) Go bindings.
- If you’re writing a Dqlite client for a new language, refer to the [wire protocol](https://dqlite.io/docs/protocol) description.

If you have questions about Dqlite, have a look at the [FAQ](/t/documentation-faq/22) before filing issues.
