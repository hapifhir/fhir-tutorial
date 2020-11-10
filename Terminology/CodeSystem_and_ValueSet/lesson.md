# Introduction to Terminology

Many elements in the FHIR resources have a coded value: some fixed string (a sequence of characters) assigned elsewhere that identifies some defined “concept”.
Using Codes in Resources (https://www.hl7.org/fhir/terminologies.html#4.1)

“coded value” = “coded element” = “concept”

## For Clarity

| General concepts  | FHIR Resources |
|-------------------|----------------|
| code system       | CodeSystem     |
| value set         | ValueSet       |

## Concept

The general pattern for representing coded elements is using the following four elements:

|          |                                                                          |
|----------|--------------------------------------------------------------------------|
| system   | A URI that identifies the system                                         |
| version  | Identifies the version of the system                                     |
| code     | A string pattern that identifies a concept as defined by the code system |
| display  | A description of the concept as defined by the code system               |


“code” ≠ “concept”

## Example of a Concept

The Coding data type represents this pattern.

```json
{
  "system" : "http://loinc.org",
  "version" : "2.62",
  "code" : "55423-8",
  "display" : "Number of steps in unspecified time Pedometer"
}
```

## Terminology Resources

|                 |                                           |
|-----------------|-------------------------------------------|
| CodeSystem - N  | A list of codes defined by some authority |
| ValueSet - N    | A list of codes for a specific purpose    |
| ConceptMap - 3* | A list of code mappings                   |

## More Terminology Resources

Out of scope:

NamingSystem - 1*

TerminologyCapabilities - 0*

## Relevant Data Types

**code** - The instance represents the code only. The system is implicit - it is defined as part of the definition of the element, and not carried in the instance.

**Coding** - A data type that has a code and a system element that identifies where the definition of the code comes from.

**CodeableConcept** - A type that represents a concept by plain text and/or one or more coding elements.

**Quantity** - The instance has system and code elements for carrying a code for the type of unit, and these can be bound to a value set.

**string** - The instance carries a string. In some cases, applications may wish to control the set of valid strings for a particular element, so the string value can be treated as a coded element (like code).

**uri** - Like string, URIs can be treated as a coded element.

# CodeSystem

## CodeSystem 

* Has an identifying URL (e.g. http://loinc.orc)
* Contains a set of codes
* May contain hierarchies
    * E.g. “HB” and “WBC” are children of “HEMO”
* May contain properties
    * E.g. loinc codes contain properties such as “STATUS” and “METHOD”
* May be defined by standards bodies (e.g. HL7 or LOINC) or by individuals

## Codes in a CodeSystem

* Each code in a system has the following things:
    * System:   http://loinc.org
    * Code:      718-7
    * Display:   Hemoglobin [Mass/volume] in Blood
* Codes may also have properties and children

## CodeSystem Required Elements

| Element Name | Type | Description                                                                  |
|--------------|------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| status       | code | Status of the resource: <br/> draft \| active \| retired \| unknown |
| content      | code | A code that indicates the extent that the content of the code system (the concepts and codes it defines) are represented in this resource instance: <br/> not-present \| example \| fragment \| complete \| supplement |

## CodeSystem Optional Elements

| Element Name     | Type            | Description                                                                  |
|------------------|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| url              | uri             | The canonical URL that never changes for this code system. This canonical URL is used to refer to all instances of this particular code system across all servers and systems. Ideally, should be a URL which resolves to the location of the master version of the code system. |
| version          | string          | Indicates a version for the CodeSystem resource.|
| versionNeeded    | boolean         | Indicates whether a version is required (i.e. if CodeSystem definition is not stable) |
| publisher        | String          | Name of the publisher for CodeSystem |
| contact          | ContactDetail   | Contact details for publisher |
| useContext       | usageContext    | The context that the content is intended to support |
| jurisdiction     | CodeableConcept | Intended jurisdiction for the code system. |
| purpose          | markdown        | Why this code system is defined. |
| copyright        | markdown        | Use and/or publishing restrictions |
| hierarchyMeaning | code            | If the code system a hierarchy of codes, describes the meaning of the relationships between parent and child codes: <br/>  grouped-by \| is-a \| part-of \| classified-with |
| compositional    | boolean         | Flag indicating whether the code system describes a compositional grammar. |
| concept          | BackboneElement | Definitions for one or more concepts that are included in the code system. Each concept component includes:<ul><li> code - a string code value</li><li> display - a string display value for the concept.</li><li> definition - a formal definition for the concept.</li><li> designation - one or more BackboneElement elements providing other representations for the concept (e.g. different languages)</li><li> property - one or more BackboneElement elements used to describe a specific property of the concept. <li>  |

## CodeSystem $lookup Operation

Given a code/system, or a Coding, get additional details about the concept, including definition, status, designations, and properties. 

| In Parameter     | Type            | Description                                                                  |
|------------------|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| code             | code            | The code to lookup. If a code is provided, a system must be provided. |
| system           | uri             | The system for the code that is to be located. |
| version          | string          | The version of the system, if one was provided in the source data. |
| coding           | Coding          | A coding to look up. |
| date             | dateTime        | The date for which the information should be returned. |
| displayLanguage  | code            | The requested language for display (language as defined in the concept designation element). |
| property         | code            | A property that the client wishes to be returned in the output. If no properties are specified, the server chooses what to return. |

## CodeSystem $validate-code Operation

Validate that a coded value is in the code system. The operation returns a result (true / false), an error message, and the recommended display for the code. 

| In Parameter     | Type            | Description                                                                  |
|------------------|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| url              | uri             | The CodeSystem URL. |
| version          | string          | The version of the system, if one was provided in the source data. |
| code             | code            | The code to be validated. |
| coding           | coding          | A coding to validate. The system must match the specified code system. |
| codeableConcept  | CodeableConcept | A full codeableConcept to validate. |
| codeSystem       | CodeSystem      | The codeSystem is provided directly as part of the request. |
| display          | string          | The display associated with the code, if provided.  |
| date             | dateTime        | The date for which the validation should be checked. 
| abstract         | boolean         | If this parameter has a value of true, the client is stating that the validation is being performed in a context where a concept designated as 'abstract' is appropriate/allowed to be used, and the server should regard abstract codes as valid. If this parameter is false, abstract codes are not considered to be valid. |

## CodeSystem $subsumes Operation

Test the subsumption relationship between code/Coding A and code/Coding B given how the concepts are defined in the underlying code system and how parent/child relationships are defined in the CodeSystem  (see hierarchyMeaning). 

Operation returns an outcome which includes one of the following codes:

* equivalent
* subsumes
* subsumed-by
* not-subsumed

| In Parameter     | Type            | Description                                                                  |
|------------------|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| codeA            | code            | The "A" code that is to be tested. |
| codeB            | code            | The "B" code that is to be tested. |
| system           | uri             | The code system in which subsumption testing is to be performed. |
| version          | string          | The version of the system, if one was provided in the source data |
| codingA          | coding          | The "A" Coding that is to be tested. |
| codingB          | coding          | The "B" Coding that is to be tested. |

