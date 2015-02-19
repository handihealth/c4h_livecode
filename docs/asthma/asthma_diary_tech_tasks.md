##C4H Asthma Diary Ehrscape Technical tasks
This document describes the series of Ehrscape API calls required for the Asthma Diary app and assumes a base level of understanding of the Ehrscape API and the use of openEHR - further details can be found at [Overview of openEHR and Ehrscape](/docs/training/openehr_intro.md).

The steps covered are...

  A. Retrieve an Ehrscape Session token   
  B. Retrieve Patient's subjectId from the demographics service, based on NHS Number  
  C. Retrieve Patient's ehrId from Ehrscape, based on their subjectId 
  D. Retrieve list of Patient's recent Asthma Diary Entry compositions  
  E. Retrieve a single Asthma Diary Encounter composition  
  F. Retrieve detailed list of recent Asthma Diary Entries for charting purposes  
	G. Persist a new Asthma Diary Encounter Composition  
	H. Other API services  
	    1. Access the ALISS 'Local community resources' service    
			2. Access the NHS Choices Patient advice service - HTML  
			3. Access the NHS Choices Patient advice service - XML    
			4. Access the Indizen SNOMED CT Terminology browser service  

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
###B. Retrieve Patient's subjectId from the demographics service based on NHS Number

An external identifier associated with the patient, in this case their NHS Number, is used to retrieve the patient's internal demographic service identifier their `subjectId`.

The root `id` element i.e. **"63436" in the JSON response example below** is the `subjectId`.

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

###C. Retrieve Patient's ehrId from Ehrscape based on their subjectId

We now need to retrieve the patient's internal `ehrID` asociated with their subjectId. The ehrID is a unique string which, for security reasons, cannot be assoicated with the patient, if for instance their openEHR records were leaked.

#####Call: Returns the EHR for the specified subject ID and namespace.
````
GET /rest/v1/ehr/?subjectId=63436&subjectNamespace=ehrscape
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
#####Return:
````json
{
  "ehrId": "d848f3b3-25a2-4eff-bd94-acfb425cf1d8"
}
````


###D. Retrieve list of Patient's recent Asthma Diary Entry compositions

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
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

The `uid_value` element in the response is the unique identifier for the composition and `context_start_time` is the time that the document was authored.

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as uid_value, a/context/start_time/value as context_start_time from EHR e[ehr_id/value='d848f3b3-25a2-4eff-bd94-acfb425cf1d8'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1] where a/name/value='Asthma Diary Entry' order by a/context/start_time/value desc offset 0 limit
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````

#####Return:
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

###E. Retrieve a single Asthma Diary Encounter composition

We will use the results of the previous query to retrieve one of the compositions via its compositionId

#####Call: Returns the specified Composition in FLAT JSON format.
````
GET /rest/v1/composition/10ade55e-75d1-4278-bf3b-7ef20bcddee8::c4h_train.ehrscape.com::1?format=FLAT
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
 Content-Type: application/json
````
#####Return:
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

###F. Retrieve detailed list of recent Asthma Diary Encounters for charting purposes

We now use a more complex Archetype Query Language (AQL) call to retrieve a the key datapoints of the most recent Asthma Diary Encounter``composition``.

AQL allows for very flexible querying and output. The output chosen here is relatively flat, returning a set of simple name/pair values. This is simple to process but does have the disadvantage of causing some duplication of returned rows to accomodate multiple symptoms. More complex AQL structures can be returned but are more difficult to process.

#####AQL statement
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

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as uid_value, a/context/start_time/value as context_start_time,     b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0057]/items[at0008, 'Best predicted result']/value/magnitude as Best_predicted_result,     b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0057]/items[at0058]/value/magnitude as Actual_Result,  b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0099, 'PEFR Score']/value/magnitude as PEFR_Score,     b_b/data[at0001]/events[at0002]/data[at0003]/items[at0127]/items[at0057]/items[at0122]/value/numerator as Actual_predicted_Ratio_numerator, b_c/items[at0001]/value/value as Symptom_name,     b_a/data[at0001]/events[at0002]/data[at0003]/items[at0004]/value/value as Comment from EHR e contains COMPOSITION a[openEHR-EHR-COMPOSITION.encounter.v1] contains (     OBSERVATION b_b[openEHR-EHR-OBSERVATION.pulmonary_function.v1] or     OBSERVATION b_a[openEHR-EHR-OBSERVATION.story.v1] or     CLUSTER b_c[openEHR-EHR-CLUSTER.symptom.v1]) where a/name/value='Asthma Diary Entry' order by a/context/start_time/value desc offset 0 limit 10  

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
#####Return:
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

###G. Persist a new Asthma Diary Encounter Composition

All openEHR data is persisted as a COMPOSITION (document) class. openEHR data can be highly structured and potentially complex. To simplify the challenge of persisting openEHR data, examples of  'target composition' data instances have been provided in the Ehrscape FLAT JSON format

Once the data is assembled in the correct format, the actual service call is very simple requiring only the setting of simple parameters and headers.

[Example Ehrscape Flat JSON Composition](/technical/instances/asthma/Asthma Diary FLAT_4.json)  

#####Call: Creates a new openEhr composition and returns the new CompositionId
````
POST /rest/v1/composition?ehrId=d848f3b3-25a2-4eff-bd94-acfb425cf1d8&templateId=C4H Asthma Diary Encounter&committerName=handi&format=FLAT

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
````json 
 {
  "ctx/composer_name": "Steve Walford",
  "ctx/health_care_facility|id": "999999-345",
  "ctx/health_care_facility|name": "Northumbria Community NHS",
  "ctx/id_namespace": "NHS-UK",
  "ctx/id_scheme": "2.16.840.1.113883.2.1.4.3",
  "ctx/language": "en",
  "ctx/territory": "GB",
  "ctx/time": "2014-09-25T09:11:02.518+02:00",
    "asthma_diary_entry/history:0/story_history/comment": "Back to normal",
    "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/test_result_name|code": "at0071",
    "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/best_predicted_result|magnitude": 630,
    "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/best_predicted_result|unit": "l/min",
    "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_result|magnitude": 630,
    "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_result|unit": "l/min",
    "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_predicted_ratio|numerator": 100,
    "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pulmonary_flow_rate_result/actual_predicted_ratio|denominator": 100.00,
    "asthma_diary_entry/examination_findings:0/pulmonary_function_testing:0/result_details/pefr_score": "Green"
}
````
#####Return:
````json
{
 "href": "https://rest.ehrscape.com/rest/v1/composition/8516f953-100f-4edb-8607-96120076f529::c4h_train.ehrscape.com::1" 
}
````

##H. Other Services


###1. Access the ALISS 'Local community resources' service

[ALISS](http://www.aliss.org/) (A Local Information System for Scotland) is a search and collaboration tool for Health and Wellbeing resources. Originally developed in Scotland, it is now us used in regions across the UK. 

Further search options e.g by locality are avaiable from the [ALISS API documentation site](http://aliss.readthedocs.org/en/latest/search_api/)

Call: Search the ALLIS database for resources related to asthma
````
GET http://www.aliss.org/api/v2/search/?q=asthma

Headers:
 None 
````
Return:
````json
{
    "count": 91, 
    "next": "http://www.aliss.org/api/v2/search/?page=2&q=asthma", 
    "previous": null, 
    "results": [
        {
            "id": 351, 
            "title": "Asthma - a booklist to help you manage the condition - from Renfrewshire Libraries", 
            "description": "Renfrewshire Libraries have developed a facility for creating booklists online via a simple search function. One can find out whether Renfrewshire Libraries has a particular title, whether it is available and from which library, etc.\r\n\r\nThis one is about Asthma.", 
            "uri": "https://libcat.renfrewshire.gov.uk/vs/List.csp?SearchT1=asthma&Index1=Keywords&Database=1&PublicationType=NoPreference&Location=NoPreference&SearchMethod=Find_1&SearchTerm1=asthma&OpacLanguage=eng&Profile=Default&EncodedRequest=*D5*2C2*D8*EE*C4*D4*5B*B2*9C*00*26*D8*5Cb*8F&EncodedQuery=*D5*2C2*D8*EE*C4*D4*5B*B2*9C*00*26*D8*5Cb*8F&Source=SysQR&PageType=Start&PreviousList=RecordListFind&WebPageNr=1&NumberToRetrieve=20&WebAction=NewSearch&StartValue=0&RowRepeat=0&ExtraInfo=SearchFromList&SortIndex=Author&SortDirection=1", 
            "locations": [
                {
                    "lat": 55.8561, 
                    "lon": -4.4057, 
                    "formatted_address": "Renfrew South & Gallowhill ward, Renfrewshire, PA3 4SF"
                }
            ], 
            "tags": [
                "books", 
                "library", 
                "Asthma", 
                "asthma"
            ], 
            "owner": "Living Well at the Library", 
            "event_start": null, 
            "event_end": null, 
            "created_on": "2011-07-13T12:46:44.732000+00:00", 
            "modified_on": "2014-04-22T07:15:37.249920+00:00"
        }, 
				...
}

````

###2. Access the NHS Choices Patient advice service - HTML

This API call retreives NHS Choices patientinformation 

#####Call: Search the NHS Choices database for resources related to asthma, returning an HTML page
````
GET http://v1.syndication.nhschoices.nhs.uk/conditions/articles/asthma/introduction?apikey=GFENGBJA

Headers:
 None 
````
#####Return:
````html
<html xmlns="http://www.w3.org/1999/xhtml" >
<head>
    <title>NHS Choices Syndication</title>
    <meta http-equiv="Content-Style-Type" content="text/css" />
.....
</head>
````

###3. Access the NHS Choices Patient advice service - XML

This API call retreives NHS Choices patient information for Asthma in XML format.

#####Call: Search the NHS Choices database for resources related to asthma, returning an XML document.
````
GET http://v1.syndication.nhschoices.nhs.uk/conditions/articles/asthma/introduction.xml?apikey=GFENGBJA

Headers:
 None 
````
#####Return:
````xml
<?xml version="1.0" encoding="utf-8"?>
     <feed xmlns="http://www.w3.org/2005/Atom"><title type="text">NHS Choices - Introduction</title><id>uuid:6e093010-1013-46b4-b231-42795514008d;id=1950</id><rights type="text">© Crown Copyright 2009</rights><updated>2015-02-11T15:54:40Z</updated><category term="asthma" /><logo>http://www.nhs.uk/nhscwebservices/documents/logo1.jpg</logo>
	...
````

###4. Access the Indizen SNOMED CT Terminology browser service

This API call to the [Indizen](www.indizen.com/index.php/en/) Terminology serviceretreives SNOMED CT terms matching ``asthma`` in XML format.

#####Call: Search the Indizen terminology service database for terms matching asthma
````
GET /ITSNode/rest/snomed/descriptions?matchvalue=asthma&referencelanguage=en&fuzzy=false&numberOfElements=110&filtercomponent=all&spellingCorrection=true 

Headers:
 Authorization: Basic aGFuZGlob3BkOmRmZzU4cGE= 
````
#####Return:
````xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<sctDescriptionss>
	<description>
		<conceptid>195967001</conceptid>
		<descriptionid>301485011</descriptionid>
		<descriptionstatus>0</descriptionstatus>
		<descriptiontype>1</descriptiontype>
		<inititalcapitalstatus>0</inititalcapitalstatus>
		<languagecode>en</languagecode>
		<similarity>100.0</similarity>
		<sourceName/>
		<term>Asthma</term>
	</description>
	<description>
		<conceptid>161527007</conceptid>
		<descriptionid>251716013</descriptionid>
		<descriptionstatus>0</descriptionstatus>
		<descriptiontype>1</descriptiontype>
		<inititalcapitalstatus>1</inititalcapitalstatus>
		<languagecode>en</languagecode>
		<similarity>98.0</similarity>
		<sourceName/>
		<term>H/O: asthma</term>
	</description>
	<description>
		<conceptid>160377001</conceptid>
		<descriptionid>249995012</descriptionid>
		<descriptionstatus>0</descriptionstatus>
		<descriptiontype>1</descriptiontype>
		<inititalcapitalstatus>1</inititalcapitalstatus>
		<languagecode>en</languagecode>
		<similarity>98.0</similarity>
		<sourceName/>
		<term>FH: Asthma</term>
	</description>
   ....
</sctDescriptionss>
````

