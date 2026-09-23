# Coverage — ask the service, do not trust a number on a page

GoldSeam is capturing college catalogs continuously. Coverage is **uneven by design of the work, not by
choice**: some states and schools are complete, others are not started, and a school with nothing held
answers **not held**.

**No coverage figure is written in this repository.** Any number typed here would be wrong within a
week. The service publishes its own, and that one is always current:

```
get_courseshelf_summary
```

It returns, as of the publish serving right now:

- **what is held** — schools identified, schools with catalog content loaded, and the record counts by area (programs, checklists, courses, credit-by-exam);
- **by state** — for each state, how many schools are held and how many carry programs, checklists and courses;
- **the census reading** — the measures the build is held to: *found* (schools loaded), *confidence* (requirement items resolved) and *full set* (schools complete across areas);
- **the build and publish** that answered, with their dates.

## How to read the census

The census is a **measure, not a gate**. It is published with the data rather than held over it, so
anyone can see how far the capture has got. A low reading in a state means the catalogs there have not
been captured yet — not that the schools there are different.

## If your school is not held

That is the most useful thing you can tell us. [Open an issue](../README.md#support) with the school's
name and, if you have it, its IPEDS UNITID. Schools people are actually asking about go first.
