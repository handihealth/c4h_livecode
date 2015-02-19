#Overview of openEHR concepts
openEHR is a open specification for an electronic health record, published and maintained by the [openEHR Foundation](http://openehr.org).

The full openEHR specification is complex and beyond the scope of thissimple overview. In summary an application-builder interacts with an an openEHR system via a simple set of APIs, and defines its information model and querying requirements via the use of shareable (and normally open-sourced) 'archetypes and templates', rather than by working directly with the database.

##Overview of openEHR Reference model
The openEHR Reference model defines a relatively small set of information model constructs which openEHR back-ends must support. This includes a number of generic classes and datatypes.

The Reference model contains virtually no clinical content e.g concepts for Medication, or Diagnosis. These are defined and managed separately as 'archetypes'

###Key openEHR datatypes
openEHR has a very rich set of allowable datatypes. A full defiinition is beyond the scope of this document but developers new to this field may find the following notes helpful.

####Datatype: text
**'text'** allows the recording of simple,unformatted text. openEHR does not normally constrain the length of string.

````
//FLAT JSON  
"asthma_diary_entry/history:0/story_history/comment": "Feeling much better",
````

####Datatype: codedText

**'codedText'** is a commonly used datatype in openEHR systems and is a sub-class of text. i.e where-ever *text* is specified *codedText* can be used instead.

Codes may be 'external' e.g. SNOMED CT or 'local', where they are defined within archetypes, have the form 'atxxxxx' and are commonly referred to as **'atCodes'**

A codedText element always includes the terminologyID, the code itself and the text of the coded concept (Rubric).  
  Where a codedText item is required, allowed value(s) are expressed in the form:  

    terminologyId::code::rubric

    local::at0007::Dental swelling
    SNOMED-CT::123456::No pathology found

When a codedText item is added to a FLAT JSON format document, you must give the code, value and terminology, unless this is a local (atCode) code,
 in which case only the code needs to be provided.

#####External terminology e.g SNOMED CT

Code, terminology and value must be specified
````
//FLAT JSON
"asthma_diary_entry/history:0/story_history/symptom:0/symptom_name|code":     "23924001",
"asthma_diary_entry/history:0/story_history/symptom:0/symptom_name|value": "chest tightness",
"asthma_diary_entry/history:0/story_history/symptom:0/symptom_name|terminology": "SNOMED-CT",
````

#####Local 'atCode' e.g at0007

Only the code needs to be specified - the value and terminology are not required since they are pre-defined in the openEHR template..

````
//FLAT JSON
"asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/test_result_name|code": "at0071"
````
####Datatype: ordinal
 **'ordinal'** is a datatype which combines codedText with a score, expressed as an integer.  

   *  0: Green  `local::at0022::Green`
   *  1: Amber  `local::at0023::Amber`
   *  2: Red    `local::at0024::Red`

````
//FLAT JSON
"community_dental_final_assessment_letter/assessment_scales/dental_rag_score:0/caries_tooth_decay/caries_risk|code": "at0024",
"community_dental_final_assessment_letter/assessment_scales/dental_rag_score:0/caries_tooth_decay/caries_risk|ordinal": 2,
"community_dental_final_assessment_letter/assessment_scales/dental_rag_score:0/caries_tooth_decay/caries_risk|value": "Red",
````

####Datatype: count
**'count'** is a simple integer.  
````
" //FLAT JSON
"community_dental_final_assessment_letter/investigations_and_results:0/imaging_examination_result:0/result_group/decayed_teeth/decayed_teeth": 4,
````

####Datatype: datetime
**'dateTime'** records a date or date and time using the [ISO8061 format](http://www.w3.org/TR/NOTE-datetime).

````
//FLAT JSON
"ctx/time": "2014-09-23T00:11:02.518+02:00"
````


####Datatype: quantity
**quantity** records a physical quantity along with the appropriate SI units, which should normally be compliant with [UCUM](http://unitsofmeasure.org).

````
//FLAT JSON
"asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_result|magnitude": 550,
"asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_result|unit": "l/min",

````

###Key openEHR concepts and classes

####External identifiers

The subjectId (sometimes called externalId) is a patient identifier by which the patient is known outwith the openEHR system e.g. a hospital identifier or NHS number.

####EHR
In an openEHR system, each patient has an ``ehr`` (a per-patient electronic health record) with a unique ``ehrId`` identifier (usually a guid). All references to openEHR patient data are via this ehrId. An API call `GET ehr/` is used to reference the ehrId from the provided subjectId/externalId.

Example ehrId: ``8fa77360-683c-4989-be5e-89a192624b43``

Other subsequent calls to the openEHR system for that particular patient are via the ehrId.

####COMPOSITION
All openEHR data is committed inside a ``composition``, a document-level container which is given a unique ``compositionId``. If the composition is subsequently updated the root compositionId remain unchanged but its version number is incremented. All previous composition versions are retained for audit/legal purposes.

Initial version
``2cf04d6c-31e8-4599-a14b-9a0add4de5d9::fivium.ehrscape.com::1``

Revised version
``2cf04d6c-31e8-4599-a14b-9a0add4de5d9::fivium.ehrscape.com::2``

If you need to retrieve a composition, it is normally ok to simply use the root part ``2cf04d6c-31e8-4599-a14b-9a0add4de5d9`` which will return the latest revision in the current repository.


##### openEHR Composition file formats

In general, committing or retrieving a composition to/from an openEHR system involves sendining or receiving a structured XML or JSON file via a simple API.

The current standard **'canonical XML'** openEHR Composition format is defined by a set of [standard schema](https://github.com/openEHR/specifications/tree/master/ITS/XML-schema).  

More recently, openEHR implementers have started to develop alternative custom JSON formats ( usuually with converters to/from the canonical XML format).

The [HANDI-HOPD Ehrscape API](https://www.ehrscape.com/api-explorer.html), developed by Marand, provides a simple restful API which hides much of the complexity of the underlying openEHR server accepting simpler, flatter forms of composition data, using defaults within the template schema to correctly populate the raw openEHR data which is stored internally.

The `FLAT JSON` format is just a set of simple name/value pairs where the 'name' carries the path to each element. You do not need to parse this path. You should normally use this **FLAT JSON** format, which is easier for new openEHR users.

e.g.

````
FLAT JSON format:
   ... "community_dental_final_assessment_letter/assessment_scales/dental_rag_score:0/caries_tooth_decay/clinical_factors|code": "at0025", ...

STRUCTURED JSON format
     ... "clinical_factors": [
       {
        "|code": "at0025",
        "|terminology": "local",
        "|value": "Teeth with carious lesions"
      }
      ]	...

RAW XML format
  ... <ns2:value xsi:type="ns2:DV_CODED_TEXT">
          <ns2:value>Teeth with carious lesions</ns2:value>
              <ns2:defining_code>
               <ns2:terminology_id>
                 <ns2:value>local</ns2:value>
              </ns2:terminology_id>
              <ns2:code_string>at0025</ns2:code_string>
             </ns2:defining_code>
          </ns2:value>
       </ns2:value> ...
````

### Key openEHR Reference model attributes

A number of key data points need to be populated in an openEHR composition, which may not be apparent from the archetypes or templates. Developers can largely use the example instance documents and APIs for guidance but these notes may give useful background in addition to viewing the [UML view of the openEHR reference model.](http://www.openehr.org/local/releases/1.0.1/uml/index.html)

**committer** This is the name of the person physically committing the document ie. the person logged on to the account. If omitted from API calls, Ehrscape will use the domain login name.

**composition/composer:** This is the clinical author of the document i.e the person with clinical responsibility. Ehrscape FLAT and STRCTURTED formats handle this as ``composer_name``.

**composition/context/start_time:** This is the time that the clinical interaction with the patient began. Ehrscape FLAT and STRUCTURED formats handle this as ctx/time.

**composition/context/health_care_facility:** This is the healthcare facility / oragnisation under who's remit the encounter took place.

**observation/time:** This is the time that a patient's signs and symptoms were observed ora test was run. It is set automatically by the value of the ctx/time attribute

The Ehrscape FLAT and STRUCTURED formats hide much of the complexity of these attributes, providing sensible defaults.
In particular the `ctx` header common to both JSON STRUCTURED and FLAT formats, considerably simplifies the composition header ...

     "ctx/composer_name": "Rebecca Wassall",
     "ctx/health_care_facility|id": "999999-345",
     "ctx/health_care_facility|name": "Northumbria Community NHS",
     "ctx/id_namespace": "NHS-UK",
     "ctx/id_scheme": "2.16.840.1.113883.2.1.4.3",
     "ctx/language": "en",
     "ctx/territory": "GB",
     "ctx/time": "2014-09-23T00:11:02.518+02:00",
