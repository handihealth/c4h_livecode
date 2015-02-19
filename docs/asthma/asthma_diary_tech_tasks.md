##C4H Asthma Diary Technical tasks
This document describes the series of API calls required to c for the Astmas Diary app.

###A. Retrieve session token

The first step in working with Ehrscape is to retrieve the session token. This allows subsequent API calls to be made without needing to login on each occasion.
The session should be formally closed when you are finished

#####Call: Create a new openEHR session:
 ````
 POST /rest/v1/session?username=c4h_train&password=c4h_train99
 ````
#####Returns:
````json
{
"sessionId": "fc234d24-7b59-49f5-a724-8d37072e832b"
}
````
###B. Retrieve Patient's subjectId from the demographics service

The subjectId is an external identifier associated with the patient - in this case their NHS Number.

The subjectId required in the response is the root `id` element i.e. **"63436"**.

#####Call: Queries the demographics store for matching parties, with the query parameters specified in the URL.
 ````
GET /rest/v1/demographics/party/query/?uk.nhs.nhsnumber=7430555
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
#####Returns:
````json
{
    "parties": [
        {
            "id": "63436",
            "version": 0,
            "firstNames": "Steve",
            "lastNames": "Walford",
            "gender": "MALE",
            "dateOfBirth": "1965-07-12T00:00:00.000Z",
            "address": {
                "id": "63436",
                "version": 0,
                "address": "60 Florida Gardens, Cardiff, LS23 4RT"
            },
            "partyAdditionalInfo": [
                {
                    "id": "63437",
                    "version": 0,
                    "key": "uk.nhs.nhsnumber",
                    "value": "7430555"
                },
                {
                    "id": "63438",
                    "version": 0,
                    "key": "title",
                    "value": "Mr"
                }
            ]
        }
    ]
}
````


###C. Retrieve Patient's ehrId from Ehrscape

We now need to retrieve the patient's internal ``ehrID`` asociated with their subjectId. The ehrID is a unique string which, for security resons, cannot be assoicated with the patient, if for instance their openEHR records were leaked.

Call: Returns the EHR for the specified subject ID and namespace.
````
GET /rest/v1/ehr/?subjectId=63436&subjectNamespace=ehrscape
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
Return:
````json
{
  "ehrId": "d848f3b3-25a2-4eff-bd94-acfb425cf1d8"
}
````


###D. Retrieve list of Patient's recent Asthma Diary Entries

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Asthma Diary encounter ``composition`` records. Compositions are document-level records which act as the container for all openEHR patient data.

The AQL statement which retrieves the compositionId for the most recent 10 Diary Entries is

````
select
    a/uid/value as uid_value,
    a/context/start_time/value as context_start_time
from EHR e[ehr_id/value='d848f3b3-25a2-4eff-bd94-acfb425cf1d8']
contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1]
where a/name/value='Asthma Diary Entry'
order by a/context/start_time/value desc
offset 0 limit 10
`````
The query API call returns a ``resultset`` which is a nested set of name/value pairs whose format is determined by the AQL query.

 Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as uid_value, a/context/start_time/value as context_start_time from EHR e[ehr_id/value='d848f3b3-25a2-4eff-bd94-acfb425cf1d8'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1] where a/name/value='Asthma Diary Entry' order by a/context/start_time/value desc offset 0 limit
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId

 data
````
Return:
````json
"resultSet": [
        {
            "context_start_time": "2014-09-23T09:11:02.518+02:00",
            "uid_value": "6466886b-1a89-4362-b253-2edb8f45d968::c4h_train.ehrscape.com::1"
        },
        {
            "context_start_time": "2014-09-22T09:11:02.518+02:00",
            "uid_value": "67d28ff7-216c-4744-9737-1d0be5e300e1::c4h_train.ehrscape.com::1"
        },
        {
            "context_start_time": "2014-09-21T21:21:02.518+02:00",
            "uid_value": "5b908abd-329b-4f7a-8e27-86fe07b141ac::c4h_train.ehrscape.com::1"
        },
        {
            "context_start_time": "2014-09-21T08:11:02.518+02:00",
            "uid_value": "57a15efd-f540-4ec5-bbac-0f5e6a0d90fd::c4h_train.ehrscape.com::1"
        },
        {
            "context_start_time": "2014-09-20T09:11:02.518+02:00",
             "uid_value": "10ade55e-75d1-4278-bf3b-7ef20bcddee8::c4h_train.ehrscape.com::1"
        }
    ]
````

###E. Retrieve Patient's Single Asthma Diary Encounter composition

We will use the previous query to retrieve the whole composition.

Call: Returns the specified Composition in FLAT JSON format.
````
GET /rest/v1/composition/10ade55e-75d1-4278-bf3b-7ef20bcddee8::c4h_train.ehrscape.com::1?format=FLAT
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
 Content-Type: application/json
````
Return:
````json
"composition": {
       "asthma_diary_entry/context/setting|238": true,
       "asthma_diary_entry/context/setting|value": "other care",
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_predicted_ratio": 0.92,
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/time": "2014-09-20T07:11:02.518Z",
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/test_result_name|code": "at0071",
       "asthma_diary_entry/context/setting|code": "238",
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/best_predicted_result|unit": "l/min",
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/test_result_name|at0071": true,
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_result|magnitude": 580,
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/test_result_name|value": "Peak expiratory flow (PEF)",
       "asthma_diary_entry/_uid": "10ade55e-75d1-4278-bf3b-7ef20bcddee8::c4h_train.ehrscape.com::1",
       "asthma_diary_entry/context/start_time": "2014-09-20T07:11:02.518Z",
       "asthma_diary_entry/context/setting|terminology": "openehr",
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pefr_score": "Green",
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/best_predicted_result|magnitude": 630,
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_result|unit": "l/min",
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_predicted_ratio|denominator": 100,
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/test_result_name|terminology": "local",
       "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_predicted_ratio|numerator": 92
}
````

###D. Retrieve list of Patient's recent Asthma Diary Entries

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Asthma Diary encounter ``composition`` records. Compositions are document-level records which act as the container for all openEHR patient data.

The AQL statement which retrieves the compositionId for the most recent 10 Diary Entries is

````
select
    a/uid/value as uid_value,
    a/context/start_time/value as context_start_time,
    b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0057]/items[at0008, 'Best predicted result']/value/magnitude as Best_predicted_result,
    b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0057]/items[at0058]/value/magnitude as Actual_Result,
    b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0099, 'PEFR Score']/value/magnitude as PEFR_Score,
    b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0057]/items[at0122]/value/numerator as Actual_predicted_Ratio_numerator,
    b_c/items[at0001]/value/value as Symptom_name,
    b_a/data[at0001]/events[at0002]/data[at0003]/items[at0004]/value/value as Comment
from EHR e
contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1]
contains (
    OBSERVATION b_b[openEHR-EHR-OBSERVATION.pulmonary_function.v1] or
    OBSERVATION b_a[openEHR-EHR-OBSERVATION.story.v1] or
    CLUSTER b_c[openEHR-EHR-CLUSTER.symptom.v1])
where a/name/value='Asthma Diary Entry'
order by a/context/start_time/value desc
offset 0 limit 10
`````
**NOTE: In this case the resultset carries some 'duplicated rows' where the composition contains multiple symptoms.**

 Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as uid_value, a/context/start_time/value as context_start_time,     b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0057]/items[at0008, 'Best predicted result']/value/magnitude as Best_predicted_result,     b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0057]/items[at0058]/value/magnitude as Actual_Result,  b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0099, 'PEFR Score']/value/magnitude as PEFR_Score,     b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0057]/items[at0122]/value/numerator as Actual_predicted_Ratio_numerator, b_c/items[at0001]/value/value as Symptom_name,     b_a/data[at0001]/events[at0002]/data[at0003]/items[at0004]/value/value as Comment from EHR e contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1] contains (     OBSERVATION b_b[openEHR-EHR-OBSERVATION.pulmonary_function.v1] or     OBSERVATION b_a[openEHR-EHR-OBSERVATION.story.v1] or     CLUSTER b_c[openEHR-EHR-CLUSTER.symptom.v1]) where a/name/value='Asthma Diary Entry' order by a/context/start_time/value desc offset 0 limit 10
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
Return:
````json
"resultSet": [
       {
           "Symptom_name": "chest tightness",
           "Comment": "Feeling much better",
           "context_start_time": "2014-09-23T09:11:02.518+02:00",
           "Actual_Result": 550,
           "PEFR_Score": null,
           "uid_value": "6466886b-1a89-4362-b253-2edb8f45d968::c4h_train.ehrscape.com::1",
           "Actual_predicted_Ratio_numerator": 87.3,
           "Best_predicted_result": 630
       },
       {
           "Symptom_name": "chest tightness",
           "Comment": "Feeling much better",
           "context_start_time": "2014-09-22T09:11:02.518+02:00",
           "Actual_Result": 491,
           "PEFR_Score": null,
           "uid_value": "67d28ff7-216c-4744-9737-1d0be5e300e1::c4h_train.ehrscape.com::1",
           "Actual_predicted_Ratio_numerator": 78,
           "Best_predicted_result": 630
       },
       {
           "Symptom_name": "shortness of breath",
           "Comment": "Feeling much better",
           "context_start_time": "2014-09-22T09:11:02.518+02:00",
           "Actual_Result": 491,
           "PEFR_Score": null,
           "uid_value": "67d28ff7-216c-4744-9737-1d0be5e300e1::c4h_train.ehrscape.com::1",
           "Actual_predicted_Ratio_numerator": 78,
           "Best_predicted_result": 630
       },
       {
           "Symptom_name": "wheezing",
           "Comment": "Slightly better",
           "context_start_time": "2014-09-21T21:21:02.518+02:00",
           "Actual_Result": 308,
           "PEFR_Score": null,
           "uid_value": "5b908abd-329b-4f7a-8e27-86fe07b141ac::c4h_train.ehrscape.com::1",
           "Actual_predicted_Ratio_numerator": 49,
           "Best_predicted_result": 630
       },
       {
           "Symptom_name": "shortness of breath",
           "Comment": "Slightly better",
           "context_start_time": "2014-09-21T21:21:02.518+02:00",
           "Actual_Result": 308,
           "PEFR_Score": null,
           "uid_value": "5b908abd-329b-4f7a-8e27-86fe07b141ac::c4h_train.ehrscape.com::1",
           "Actual_predicted_Ratio_numerator": 49,
           "Best_predicted_result": 630
       },
       {
           "Symptom_name": "coughing",
           "Comment": "Slightly better",
           "context_start_time": "2014-09-21T21:21:02.518+02:00",
           "Actual_Result": 308,
           "PEFR_Score": null,
           "uid_value": "5b908abd-329b-4f7a-8e27-86fe07b141ac::c4h_train.ehrscape.com::1",
           "Actual_predicted_Ratio_numerator": 49,
           "Best_predicted_result": 630
       },
       {
           "Symptom_name": "shortness of breath",
           "Comment": "Think I may have the flu - started Prednisolone",
           "context_start_time": "2014-09-21T08:11:02.518+02:00",
           "Actual_Result": 289,
           "PEFR_Score": null,
           "uid_value": "57a15efd-f540-4ec5-bbac-0f5e6a0d90fd::c4h_train.ehrscape.com::1",
           "Actual_predicted_Ratio_numerator": 46,
           "Best_predicted_result": 630
       },
       {
           "Symptom_name": null,
           "Comment": null,
           "context_start_time": "2014-09-20T09:11:02.518+02:00",
           "Actual_Result": 580,
           "PEFR_Score": null,
           "uid_value": "10ade55e-75d1-4278-bf3b-7ef20bcddee8::c4h_train.ehrscape.com::1",
           "Actual_predicted_Ratio_numerator": 92,
           "Best_predicted_result": 630
       }
   ]
}
````


###C. Persist a new Asthma Diary Entry

All openEHR data is persisted as a COMPOSITION (document) class. openEHR data can be highly structured and potentially complex. To simplify the challenge of persisting openEHR data, examples of Dental Assessment 'target composition' data instanceshave been provided in different formats. HANDI-HOPD EhrScape will accept any of these formats and the data is in each example is persisted identically. Other openEHR servers such as OceanEHR, Code24 and Cabolabs currently only support the RAW XML format.

Once the data is assembled in the correct format, the actual service call is very simple requiring only the setting of simple parameters and headers.

**Example openEHR target Composition Document**

 [Flat JSON]()  


A typical CURL to POST (persist) FLAT JSON is ..

  POST /rest/v1/composition?ehrId=7df3a7f8-18a4-4602-a7c6-d77db59a3d23&templateId=UK AoMRC Community Dental Final Assessment&committerName=c4h_Committer&format=FLAT HTTP/1.1
  Host: rest.ehrscape.com
  Content-Type: application/json
  Authorization: Basic YzRoX2RlbnRhbDpiYWJ5dGVldGgyMw==
  Cache-Control: no-cache
  {
    "ctx/composer_name": "Rebecca Wassall",
    "ctx/language": "en",
    "ctx/territory": "GB",
    "ctx/time": "2014-09-23T00:11:02.518+02:00",
     "community_dental_final_assessment_letter/assessment_scales/dental_rag_score:0/caries_tooth_decay/caries_risk|code": "at0024",
   ...


   ##Ehrscape domains and Marand EhrExplorer

   Ehrscape is set up as a set of 'domains', each of which is populated with its own dummy patient data and openEHR templates.

   Each Ehrscape domain has its own login and password.

   There are 2 methods of accessing the ehrscape API:

   1. Set up a sessionID variable via the GET /session call. This returns a token that can be supplied in the headers of any subsequent API calls.

   2. Create a Basic Authentication string and pass this to each API call.

   The ehrscape Domain that is being used for C4H Training is

       login: ch4_train
       password: ch4_train99

       Basic Authentication: Basic YzRoX3RyYWluOmM0aF90cmFpbjk5

   #### Marand EhrExplorer

   The Marand Ehr Explorer tool is primarily used to create openEHR queries which will not be used much in this challenge but it may be helpful in allowing you to examine the openEHR templates which define the App chalenge dataset.

   1. Browse to [EHRExplorer](https://dev.ehrscape.com/explorer/)
   2. Enter the name and password for the Code4Health Dental domain `login:c4h_dental pwd:babyteeth23`
   3. Leave the domain field as `'ehrscape'`

   ![EHRExplorer](./img/ehr_explorer.png)
   In the navigation bar on the left you can double-click on the `C4H Asthma Diary Encounter` template, which will open the template in the bottom-right panel. Right-click on nodes for further details of the constraints applied.

   e.g. If you navigate to **'Plaque Control'** and right-click you can examine the internal *'atcode'* constraints allowed for this element.

   #### Ehrscape API browser

   The Marand Ehr Explorer tool is primarily used to create openEHR queries which will not be used much in this challenge but it may be helpful in allowing you to examine the openEHR templates which define the App chalenge dataset.

   1. Browse to [EHRScape API Explorer](https://dev.ehrscape.com/api-explorer.html)
   2. Press the ![Settings button](./img/tool_button.png) tool setting button and set userName to `c4h_train` and password to `c4h_train99`
   3. Any test calls in the API browser will now work against the Code4health Dental App challenge domain.


   #### Using Postman to test API calls

   The [Postman add-on](http://getpostman.com) is vey useful for testing API calls outside of an application context.
   Postman collection and environment files are available for the C4H training environment.


   ### Some basic openEHR/Ehrscape concepts

   The HANDI-HOPD Ehrscape API consumes, retrieves and queries patient healthcare data using a standardised specification and querying format defined by the openEHR Foundation. openEHR is complex and can be difficult for novices to understand (even those with a solid technical background) but the Ehrscape API considerably simplifies the interface with openEHR systems.

   **Clinical Information components**
   [openEHR](http://openehr.org) provides a way for clinicians to define and share open-source, vendor-neutral clinical information components ('archetypes' and 'templates') which can be consumed, persisted and queried by different technology stacks, as long as they adhere to the openEHR specifications. Examples of archetypes used in this project are `'Procedure', 'Symptom', and 'Imaging result'`. These are managed by the openEHR Foundation using the [Clinical Knowledge Manager](http://openehr.org/ckm) tool and mirrored to [Github](https://github.com/openEHR/CKM-mirror), with a CC-BY-SA licence.

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
