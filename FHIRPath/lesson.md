# FHIRPath: JsonPath/XPath for Fhir!

## What is it
FHIRPath is a standard by which a formal representation of logic can be applied to FHIR Resources in a reproducible manner. It is used for both resource navigation and extraction of data from resources. FHIRPath allows you to programmatically retrieve only portions of data you care about, or portions that fulfill some given criteria. All official implementations are listed [here](https://wiki.hl7.org/index.php?title=FHIRPath_Implementations), and if you want a playground to play around in, you can check out [this online tool](https://hl7.github.io/fhirpath.js/). If you are familiar with JSONPath or XMLPath, you are already halfway towards understanding the abilities and purpose of FHIRPath. 

## Resource Navigation

FHIRPath can be used for navigating through resources. Consider the following Patient
```json
{
        "resourceType": "Patient",
        "id": "1553",
        "name": [
          {
            "family": "Smith",
            "given": [
              "John",
              "Scott"
            ]
          }
        ],
        "telecom":  [
            {
              "use": "home",
              "system": "email"
            },
            {
              "use": "work",
              "value": "(1) 226 791 5555",
              "system": "phone"
            }
    }
```
To use FHIRPath to list the Patient's given names, the query would look like this: 
```bash
> Patient.name.given
{"John", "Scott"}
```
The FHIRPath evaluator will return the requested query as a collection of elements. Note that it is always a collection that is returned, even if it is only a single element:
```bash
> Patient.name.family
{"Smith"}
```
This may seem like an odd thing to do; The reason for this is in the documentation: 
```
Collection-centric: FHIRPath deals with all values as collections, allowing it to easily deal with information models with repeating elements.
```
Since everything is a collection, you can safely call collection methods on the results of a FHIRPath query: 
```bash
> Patient.name.given.first()
{"John"}
```
Some examples of such methods are `first()`, `last()`,`tail()`,`skip()`, and `children()`. Documentation for these can be found in the [functions documentation](http://hl7.org/fhirpath/#functions).
While everything in FHIRPath evaluation is a collection, it also does singleton collection evaluation. This allows you to perform the following expressions without having to select precisely which index of value you want: 
```bash
> Patient.name.family + " " + Patient.name.family
{"Smith Smith"}
```
`Warning`: This only works when the elements are actually singletons. In the following case, the FHIRPath evaluator will throw an error: 
```bash 
> Patient.name.given + " " + Patient.name.family
ERROR
```
This happens since `Patient.name.first` is not a singleton collection, but contains two elements. To correct this, you need to specify which index of the collection you want to use. 
```bash
> Patient.name.given[0] + " " + Patient.name.family
{"John Smith"}
> Patient.name.given.last() + " " + Patient.name.family
{"Scott Smith"}
```

## Polymorphism in navigation and selection
Oftentimes collections of data in a FHIR resource are not all of the same type. For example, an `Observation` may have several type options for  `value`. It could potentially be `Quantity`, or may be `boolean`, and a host of other options. Consider the following Query: 
```
> Observation.value.unit #This will return units for all value types.
> Observation.value.ofType(Quantity).unit #This will return units only if the value is a Quantity
```
`ofType()` will add only elements matching the type to the returned collection. 

## Expressions

When you query a FHIRPath evaluator, what you are passing it is called an `Expression`. FHIRPath expressions can consist of paths, literals, operators, and function invocations,

* Paths are things like `Patient.name.given`, `Observation.value`
* Literals are things like `'a'`, `true`, `0`, `@2020-02-04`
* Operators are things like `/`, `>`, `|`
* Function invocations are things like `.where()` , `.count()` , `.substring()`

```bash
> Observation.value is Quantity
true
```

### Filtering and projection of results in expressions
There are cases where you may not want _all_ of a collection. For example, what if you want all of a Patient's Telecom nodes, but only the ones which have a value of `system=phone` (e.g. you only want their phone numbers: 
```bash
> Patient.telecom.where(system='phone').value
{"(1) 226 791 5555"}
```
the `.where()` function will be called on each element in the collection, and add it to the resulting collection if the criteria in the `.where()` evaluates to true.

Similarly, if you have a collection of elements and want to select with criteria from it, you can use the projection function `select()` in order to select from the collection stream. For example, imagine for a moment that our patient from above had another name on top of their first name entry: 


```bash
> Bundle.entry.select(resource as Patient)
# This returns a collection of patients that were entries in the bundle, and ignores all non-patient resources.

> Bundle.entry.select((resource as Patient).telecom.where(system = 'phone'))
# This returns a collection of telecoms where system=phone from each resource in the bundle that was a patient.
```
Now let's imagine our patient from earlier had a second name on top of their first one, like so: 
```json 
{
    "family": "Smith2",
    "given": [
        "Frank"
    ]
}
```

### Crossing Resource Boundaries
Occasionally there are contained resources held within a given resource. Normally, FHIRPath cannot resolve these automatically, so there is a function in the spec called `resolve()` which will allow you to resolve a contained resource. The actual implementation and functionality is up to the FHIRContext evaluating the request, so it is not strictly defined in the spec. However, here is a sample query showing the usage of resolve: 
```bash
CareTeam.member.resolve().ofType(Practitioner).name.given.first()
```
This query returns a collection of all the first given names of all the Practitioners that are members of a CareTeam. 


Consider this query which will generate a collection of two strings, each representing a full name from a name element.
```bash
> Patient.name.select(given.first() + ' ' + family)
{"John Smith", "Frank Smihth2"}

```

### Aggregations

You can write general purpose aggregations, which follow the same rules as other functions, but you also have access to the special $total variable while inside. The signature of such a method is `.aggregate(aggregator: expression [, init: value])`
For example, this will generate a sum of all the data of all SampledData values in all observations in a bundle. 
```bash
Bundle.entry
.select(resource as Observation).value
.ofType(SampledData)
.aggregate($this.data + $total, 0)

```














