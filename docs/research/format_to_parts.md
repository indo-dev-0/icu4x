Format to Parts in ICU4X
========================

ECMA-402 specifies *[formatToParts](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/formatToParts#Description)*, a variant of the *format* function that returns the string split into segments according to what the segment represents.  For example:

```javascript
new Intl.DateTimeFormat("fr").format(new Date())
// "19/11/2020"

new Intl.DateTimeFormat("fr").formatToParts(new Date())
/* [
  {type: "day", value: "19"},
  {type: "literal", value: "/"},
  {type: "month", value: "11"},
  {type: "literal", value: "/"},
  {type: "year", value: "2020"},
] */

new Intl.DateTimeFormat("fr", { dateStyle: "medium" }).formatRange(new Date(2020, 0, 1), new Date(2020, 2, 3))
// "1 janv. – 3 mars 2020"

new Intl.DateTimeFormat("fr", { dateStyle: "medium" }).formatRangeToParts(new Date(2020, 0, 1), new Date(2020, 2, 3))
/* [
  {type: "day", value: "1", source: "startRange"},
  {type: "literal", value: " ", source: "startRange"},
  {type: "month", value: "janv.", source: "startRange"},
  {type: "literal", value: " – ", source: "shared"},
  {type: "day", value: "3", source: "endRange"},
  {type: "literal", value: " ", source: "endRange"},
  {type: "month", value: "mars", source: "endRange"},
  {type: "literal", value: " ", source: "shared"},
  {type: "year", value: "2020", source: "shared"},
] */
```

The goal of this document is to discuss how we will deal with formatToParts in ICU4X, both with regard to the API model and the data model (implementation).

## API Model

There are two prevailing models in formatting to parts: linear attributes and nested fields.

### Linear Attributes

ECMA-402 uses the linear attributes model: each substring is mapped to exactly one bag of attributes.  Expressed in terms of starts and ends of ranges:

```
1 janv. – 3 mars 2020

[) day, startRange
 [) literal, startRange
  [    ) month, startRange
       [  ) literal, shared
          [) day, endRange
           [) literal, endRange
            [   ) month, endRange
                [) literal, shared
                 [   ) year, Shared
```

### Nested Fields

In the nested fields model, there is a set of ranges, and a particular character could live in more than one range.  Here is how the above date interval format would look with nested fields:

```
1 janv. – 3 mars 2020

[) day
 [) literal
  [    ) month
[      ) startRange
       [  ) literal
       [  ) shared
          [) day
           [) literal
            [   ) month
          [     ) endRange
                [) literal
                 [   ) year
                [    ) shared
```

The linear attributes model is simpler, but the nested fields model is more flexible.

## Data Model

This section discusses alternate representations of formatToParts data with respect to the internal representation in types such as FormattedDateTime.

### Model A

Keep two parallel arrays, one with code points and the other with a 1-char field identifier (which could be an integer instead of a char).

```
chars  = "19/11/2020"
fields = [dd0mm0yyyy]
```

To convert from this data model to the formatToParts return value, coalesce adjacent field identifiers into ranges.  In the above example, there are 5 ranges:

1. d \[0,2)
2. 0 \[2,3)
3. m \[3,5)
4. 0 \[5,6)
5. y \[6,10)

Pros:

- **Efficient:** Code size and performance are not substantially affected by the addition of the parallel fields array to FormattedDateTime. For example, substrings can be inserted into the FormattedDateTime without having to update a separate index-based structure.
- **Linear:** The data model directly corresponds to a 1-to-1 linear mapping between code points and fields.

Cons:

- **Doesn't support multiple attributes or nested fields:** Additional business logic is required to support all bust the most basic field logic.  For example, "2020" is a year, a number, and an integer, but this data model is only able to represent one type.  This problem comes up in situations like DateIntervalFormat, where ECMA-402 requires the implementation to return multiple attributes on a particular formatToParts object.
- **Doesn't distinguish adjacent fields:** Since adjacent field identifiers are coallesced, this data model doesn't support situations where two fields of the same type are adjacent to each other in the string.  For example, in Chinese, no separators are used in certain types of lists, so different list items cannot be distinguished without an extra data structure.

### Model B

Keep a chars array plus a second array that contains field startpoints and endpoints.

```
chars = "19/11/2020"
fields = [
  { field: 'd', start: 0, end: 2 },
  { field: '0', start: 2, end: 3 },
  { field: 'm', start: 3, end: 5 },
  { field: '0', start: 5, end: 6 },
  { field: 'y', start: 6, end: 10 },
]
```

Pros:

- **Generalizable:** Supports nested fields, ability to add metadata to fields, and ability to have adjacent fields.

Cons:

- **Implementation difficulty:** This model works when the only operation is to append to the end of a string, but in i18n formatting, prepend and insert are common operations that require extra care with this data model.  Imposing this restriction on ICU4X would mean a choice between a quadratic field updating algorithm on all formatting operations, or complicated feature-flagging to disable the field structure when it is not needed.
- **Decoupled memory growth:** The chars and fields arrays in this model grow at different rates, making it harder to reason about situations where memory allocation is required.

### Model C

Use the ECMA-402 data model within the implementation.

```
parts = [
  { field: 'd', str: "19"},
  { field: '0', str: "/" },
  { field: 'm', str: "11" },
  { field: '0', str: "/" },
  { field: 'y', str: "2020" },
]
```

Pros:

- **Easy to implement:** Prepending and inserting strings is trivial.


Cons:

- **Harmful to the normal use case:** formatToParts is the exception, not the rule.  With this model, we either need extra computation to render the parts array to a string, or keep two separate string and parts structures in memory, which brings back the cons of Model B.

### Model D

Keep two parallel arrays, one with code points and the other with structured [BIES](https://www.researchgate.net/figure/Position-tags-in-a-word-BIES-tags-Tag-Description_tbl1_250723695) (begin, inside, end, single) annotations.

```
chars = "19/11/2020"
fields = [
  { field: 'd', bies: 'b' },
  { field: 'd', bies: 'e' },
  { field: '0', bies: 's' },
  { field: 'm', bies: 'b' },
  { field: 'm', bies: 'e' },
  { field: '0', bies: 's' },
  { field: 'y', bies: 'b' },
  { field: 'y', bies: 'i' },
  { field: 'y', bies: 'i' },
  { field: 'y', bies: 'e' },
]
```

Pros:

- **Efficient:** Same as Model A
- **Linear:** Same as Model A
- **Generalizable:** Unlike Model A, the structure can represent distinct adjacent fields, and nested fields can be represented by making smaller linear segments with more specific field types.

Cons:

- **Less intuitive:** Potentially less clear than the other options to ICU4X developers.