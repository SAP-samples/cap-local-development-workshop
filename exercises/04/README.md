# Exercise 04 - local messaging and events

In [The Art and Science of CAP] series Daniel Hutzel shared with us many of the influences that informed CAP's design, and explained in great detail some of the core axioms. There are key philosophical truths that are inherent in what CAP is, two of which are:

- [Everything is a service]
- [Everything is an event]

In this exercise we'll explore [events and messaging] in CAP, and in particular, the facilities in this space that are available to us when developing locally.

The general idea, as you might expect, is that CAP's eventing is agnostic at the definition and API level; the actual mechanism used to manage the receipt, queueing and relaying of messages is an implementation and platform context detail.

Whether the "message channel" facilities are provided by SAP Cloud Application Event Hub, SAP Event Mesh, or another mechanism, is largely irrelevant from a developer perspective, especially in a local context, where (in addition to an in-process facility), [file-based messaging] is available and the main focus of this exercise.

## Take a quick look at in-process eventing

Like [in-process mocking of required services], CAP supports in-process eventing. It's worth taking a brief look here, not least to have the chance to practice embracing the cds REPL, and to see how simple things are.

### Start the cds REPL

Before you begin, stop any CAP server processes for now. Then launch the REPL:

```bash
cds repl
```



---

## Further reading

---

## Footnotes

<a name="footnote-1"></a>
### Footnote 1



[The Art and Science of CAP]: https://qmacro.org/blog/posts/2024/12/06/the-art-and-science-of-cap/
[Everything is a service]: https://qmacro.org/blog/posts/2024/12/10/tasc-notes-part-4/#everything-is-a-service
[Everything is an event]: https://qmacro.org/blog/posts/2024/11/07/five-reasons-to-use-cap/#:~:text=Everything%20is%20an%20event
[events and messaging]: https://cap.cloud.sap/docs/guides/messaging/
[in-process mocking of required services]: ../03/README.md#footnote-1
[file-based messaging]: https://cap.cloud.sap/docs/guides/messaging/#_1-use-file-based-messaging-in-development
