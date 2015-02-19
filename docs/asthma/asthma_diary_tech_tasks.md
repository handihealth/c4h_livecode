##C4H Asthma Diary Ehrscape Technical tasks
This document describes the series of Ehrscape API calls required for the Asthma Diary app.

This document assumes a base level of understanding of the Ehrscape API and the use of openEHR - further details can be found at [Overview of openEHR and Ehrscape](/docs/training/openehr_intro.md).

###A. Retrieve an Ehrscape session token

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

###E. Persist a new Asthma Diary Entry

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


