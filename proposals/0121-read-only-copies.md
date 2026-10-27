# 0121: Copies that keep received data read-only

- Issue: [#121](https://github.com/itinera-dev/spec/issues/121)
- Tier: 1
- Amends: [0113](0113-value-conversions.md), [0027](0027-received-data-read-only.md) and [0091](0091-adapters-read-the-data-bag.md)

## Summary

A language may keep received data read-only by giving each receiver its own copy, as chapter 4, 4.7 point 4 allows. On some platforms, making that copy calls code the application wrote: the constructor of a record or a data class, or the equality and hashing of a set's elements or a map's keys. This proposal says such a copy is not a value conversion, so it needs no capability, says it is made for each receiver, and lists it in chapter 5, 5.10, with outcomes: a copy that fails while a value is captured means it is not a value, and one that fails while a value is read means the value is of the wrong type for that request, as for a value conversion.

Amends proposal 0113, whose definition of a value conversion could be read to include such a copy, proposal 0027, which leaves how received data is kept read-only to each language, and proposal 0091, whose reads through an input adapter's access to the data bag may now call a copy. In the chapters it changes 1.3, 1.8, 3.8, 4.2, 4.7, 5.1, 5.10 and 5.11.

## Motivation

1. **Copies call application code on some platforms.** On a platform without ownership, such as the JVM, an executor keeps 4.7 by copying values. Plain data can be copied mechanically, field by field, but a record or a data class can only be rebuilt through its constructor, which may validate what it is given, and a set or a map is rebuilt by hashing its elements or keys.
2. **Such a copy is not a conversion, but the text could be read so.** Chapter 1, 1.3 defines a value conversion as code the application supplies that turns its objects into values or values into its types, and 5.11 point 4 says a language claims `value-conversions` when its executor calls such code. Read literally, a constructor called to rebuild a copy could count, although the object keeps its type and content. A language in that position could not pass the capability's cases, which need a conversion that can refuse a value.
3. **Nothing says what happens when the copy fails.** 5.10 lists every call into custom code and says the list is exhaustive for tier 1, but it does not list a copy. A constructor that throws while a value is copied could be read as an abnormal termination, `step could not be built` or `not a value`, and two languages could choose differently.
4. **One copy per receiver.** A single read-only copy shared by every receiver would make a step that changes its input fail, by throwing, where another language lets it change its own copy. The case "A step changing its input does not change what a later step receives" of proposal 0027 expects the step and the journey to succeed. So a copy that calls application code is made for each receiver.

## Design

1. **A copy is not a value conversion.** Code the application supplies that the implementation calls only to copy a value, keeping its type and its content, so that a receiver cannot change what another receives (4.7), is not a value conversion (1.3). Examples are the constructor of a record or a data class rebuilt with copied components, and the equality and hashing of a set's elements or a map's keys. A language whose executor calls application code only to copy values does not claim `value-conversions` for it (5.11 point 4). A language MAY call both kinds of code, and then claims the capability for its value conversions.
2. **A copy is made for each receiver.** A copy that calls application code MUST be made while a value is captured, or for each request, or read through an input adapter's access to the data bag, that reads it, and every receiver MUST be able to change its own copy, where the language allows a change at all, without changing the data bag or what any other receiver gets, as 4.7 point 2 requires. A copy MUST NOT be shared between receivers in a way that turns a change into a failure.
3. **It is custom code, listed in 5.10.** The table of 5.10 point 2 gains two rows:
   1. a copy (4.7), while an object is captured as a value: it is not a value, with the outcomes of the row for a value conversion that fails while an object is captured, and for initial data or initial global metadata, a refusal listing the key;
   2. a copy (4.7), while a request reads a value: the value is of the wrong type for that request, with the outcomes of the row for a value conversion that fails while a request reads a value, including, for a read through an input adapter's access to the data bag, that the adapter is told the value is of another type and nothing is emitted.

   A copy fails by throwing or by returning an error. As for a value conversion, the abort or refusal MAY carry the error, as a diagnostic that changes neither the abort reason nor its details and that conformance does not check.
4. **A copy whose content differs** from its original is the application's mistake, which the specification does not detect: the copy is what the receiver gets.
5. **Nothing else changes.** A language that stores values as they are, or copies them without calling application code, is unaffected.

## Events

None added or changed. A copy that fails while a value is captured emits what something that is not a value emits, and one that fails while a value is read emits what a value of the wrong type emits.

## Conformance

The cases cannot name, in language-neutral terms, an object whose copy calls application code, and only a language that copies through application code could make one fail, which is why proposal 0113 needed a capability for its own cases. The outcomes are those of existing rows of 5.10, which the cases of proposals 0113 and 0056 already check. So this proposal adds no scenario and changes none: a language that copies through application code proves the two rows in its own test suite, and lists the tests in its tech spec.

## Alternatives considered

1. **One frozen copy shared by every receiver.** It makes the copy once per capture instead of once per read, but a step that changes its input would fail on the read-only copy, an abnormal termination, where another language lets it change its own copy. Rejected: the same workflow would behave differently by language.
2. **Claim `value-conversions`.** Not open to a language that only copies, since the capability's cases need a conversion that can refuse a value.
3. **Call no application code at all,** accepting only data whose collections are already read-only. Rejected as a rule: it makes the most common data types awkward to use, and it is each language's choice anyway; this proposal does not forbid it.
4. **Convert to a neutral form and back,** such as JSON, for every copy. Rejected as the suggested approach: it is a value conversion, covered by proposal 0113, and costs more than a direct copy; a language may still do it, and then claims `value-conversions`.
5. **Detect a copy whose content changed.** Rejected: it means comparing each copy with its original, which calls more application code on every copy, and conformance could not check it.

## Open questions

None.
