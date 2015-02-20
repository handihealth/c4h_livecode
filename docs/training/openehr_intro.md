#Overview of openEHR and Ehrscape
openEHR is a open specification for an electronic health record, published and maintained by the [openEHR Foundation](http://openehr.org).

The full openEHR specification is complex and beyond the scope of thissimple overview. In summary an application-builder interacts with an an openEHR system via a simple set of APIs, and defines its information model and querying requirements via the use of shareable (and normally open-sourced) 'archetypes and templates', rather than by working directly with the database.

##Ehrscape domains and Marand EhrExplorer

   The [Marand Ehrscape](https://dev.ehrscape.com/) openEHR server is set up as a set of 'domains', each of which is populated with its own dummy patient data and openEHR clinical models.

   Each Ehrscape domain has its own login and password.

   There are 2 methods of accessing the ehrscape API:

   1. Set up a sessionID variable via the GET /session call. This returns a token that can be supplied in the headers of any subsequent API calls.

   2. Create a Basic Authentication string and pass this to each API call.

   The ehrscape Domain that is being used for C4H Training is

       login: ch4_train
       password: ch4_train99

       Basic Authentication: Basic YzRoX3RyYWluOmM0aF90cmFpbjk5
			 
			 The baseURL for all Ehrscape API calls is https://rest.ehrscape.com
			 

### Marand EhrExplorer

The Marand EhrExplorer tool is primarily used to create openEHR queries which will not be used much in this challenge but it may be helpful in allowing you to examine the openEHR templates which define the App chalenge dataset.

   1. Browse to [EHRExplorer](https://dev.ehrscape.com/explorer/)
   2. Enter the name and password for the Code4Health Training domain `login:c4h_train pwd:ch4_train99`
   3. Leave the domain field as `'ehrscape'`

![EHRExplorer](/img/ehr_explorer.png)
In the navigation bar on the left you can double-click on the `C4H Asthma Diary Encounter` template, which will open the template in the bottom-right panel. Right-click on nodes for further details of the constraints applied.

e.g. If you navigate to **'Plaque Control'** and right-click you can examine the internal *'atcode'* constraints allowed for this element.

#### Ehrscape API browser

The Marand Ehr Explorer tool is primarily used to create openEHR queries which will not be used much in this challenge but it may be helpful in allowing you to examine the openEHR templates which define the App chalenge dataset.

   1. Browse to [EHRScape API Explorer](https://dev.ehrscape.com/api-explorer.html)
   2. Press the ![Settings button](./img/tool_button.png) tool setting button and set userName to `c4h_train` and password to `c4h_train99`
   3. Any test calls in the API browser will now work against the Code4health Dental App challenge domain.


#### Using Postman to test API calls

The [Postman add-on](http://getpostman.com) is vey useful for testing API calls outside of an application context.
[Postman collection and environment files](/technical/postman) are available for the C4H training environment.


##openEHR Archetypes and Templates - Clinical Information components
he HANDI-HOPD Ehrscape API consumes, retrieves and queries patient healthcare data using a standardised specification and querying format defined by the openEHR Foundation. openEHR is complex and can be difficult for novices to understand (even those with a solid technical background) but the Ehrscape API considerably simplifies the interface with openEHR systems.

[openEHR](http://openehr.org) provides a way for clinicians to define and share open-source, vendor-neutral clinical information components ('archetypes' and 'templates') which can be consumed, persisted and queried by different technology stacks, as long as they adhere to the openEHR specifications. Examples of archetypes used in this project are `'Procedure', 'Symptom', and 'Imaging result'`. These are managed by the openEHR Foundation using the [Clinical Knowledge Manager](http://openehr.org/ckm) tool and mirrored to [Github](https://github.com/openEHR/CKM-mirror), with a CC-BY-SA licence.

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

````
"ctx/composer_name": "Rebecca Wassall",
"ctx/health_care_facility|id": "999999-345",
"ctx/health_care_facility|name": "Northumbria Community NHS",
"ctx/id_namespace": "NHS-UK",
"ctx/id_scheme": "2.16.840.1.113883.2.1.4.3",
"ctx/language": "en",
"ctx/territory": "GB",
"ctx/time": "2014-09-23T00:11:02.518+02:00",
````		 
		 
###Handling specific openEHR datatypes

**text**  

Text handling is normally straightforward.

FLAT + STRUCTURED

 "synopsis": [
		        "Significant dental issues."
  ]

**codedText**  
For an external terminology, the terminologyId, code and text value must be supplied but in JSON FLAT and STRUCTURED formats only the local 'atcode' needs to be supplied.

````
		 STRUCTURED JSON format

		   Internal (local) code:
		   "dental_swelling": [
		           {
		             "|code": "at0006",
		           }
		     ]

		     External terminology:
		   "symptom_name": [
		       {
		         "|code": "102616008",
		         "|terminology": "SNOMED-CT",
		         "|value": "Painful mouth"
		       }
		     ]

		   FLAT JSON format

		   Internal (local) code:
		   "community_dental_final_assessment_letter/examination_findings:0/physical_examination_findings:0/oral_examination/dental_swelling|code": "at0006"

		   External terminology:
		     "community_dental_final_assessment_letter/history:0/story_history:0/symptom:0/symptom_name|value": "Painful mouth",
		   "community_dental_final_assessment_letter/history:0/story_history:0/symptom:0/symptom_name|code": "102616008",
		   "community_dental_final_assessment_letter/history:0/story_history:0/symptom:0/symptom_name|terminology": "SNOMED-CT"
````

**ordinal**  
For JSON FLAT and STRUCTURED formats only the local 'atcode' needs to be supplied although the ordinal and text value cacomplete are also accpeted

````
		 FLAT JSON format
		   "community_dental_final_assessment_letter/assessment_scales/dental_rag_score:0/caries_tooth_decay/caries_risk|code": "at0024"

		 STRUCTURED format

		   "caries_risk": [
		       {
		         "|code": "at0024",
		       }
		   ]

		   or

		   "caries_risk": [
		       {
		         "|code": "at0024",
		         "|ordinal": 2,
		         "|value": "Red"
		       }
		   ]
````

**date**  
Dates need to be persisted in the [ISO8061 format.](http://www.w3.org/TR/NOTE-datetime) and should be displayed in CUI format e.g. 12-Nov-1958


### Tricky issues

**Converting UI checkboxes to/from codedText**  

In a number of places, the UI may best be represented as a set of checkboxes, while the underlying data is modelled as codedText.

e.g. Symptoms

While it may seem more easier and more logical to use a boolean datatype, this is a common pattern in openEHR datasets which are designed to be interoperable and extensible. Experience has shown that exapnsion of the target valueset and alignment to external terminologies is easier if an enumerated list of codedText is used rather than boolean.

In the case of 'Symptom' the rule is ...

    If the checkbox is ticked, populate the Symptom name with the SNOMED-CT term
    If the checkbox is unticked, omit the Symptom name element completely.

Conversely when loading a persisted dataset, the checkbox should only be checked if the Symptom name element is present and contains SNOMED-CT term 102616008.

**Multiple occurrence data**  

Some aspects of the form e.g Symptoms are handled as multiple occurences of the same data point in the underlying dataset.
		 
