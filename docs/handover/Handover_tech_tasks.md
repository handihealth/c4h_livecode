##C4H Handover Summary Ehrscape Technical tasks
This document describes the series of [Ehrscape API](https://dev.ehrscape.com/api-explorer.html) calls required for the Handover Summary app and assumes a base level of understanding of that API and the use of openEHR - further details can be found at [Overview of openEHR and Ehrscape](/docs/training/openehr_intro.md).

The steps covered are...  

* Open the Ehrscape Session
* Retrieve Patient's demographics from a FHIR demographics service, based on NHS Number
* Retrieve Patient's subjectId from the local demographics service based on NHS Number
* Retrieve Patient's ehrId from Ehrscape, based on their subjectId

* Retrieve list of Patient's recent Handover Summary compositions
* Retrieve a single Handover Summary composition
* Retrieve detailed list of recent Handover Summary compositions for charting purposes
* Persist a new Handover Summary Composition
* Close the Ehrscape Session

* Other API services
 * Access the ALISS 'Local community resources' service
 * Access the NHS Choices Patient advice service - HTML
 * Access the NHS Choices Patient advice service - XML
 * Access the Indizen SNOMED CT Terminology browser service  

The baseURL for all Ehrscape API calls is https://rest.ehrscape.com


###A. Open the Ehrscape session

The first step in working with Ehrscape is to open a Session and retrieve the ``sessionId`` token. This allows subsequent API calls to be made without needing to login on each occasion.  
The session should be formally closed when you are finished.

The baseURL for all Ehrscape calls is https://rest.ehrscape.com

i.e. the call below should be to https://rest.ehrscape.com/rest/v1/session?username=c4h_train&password=c4h_train99

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
###B. Retrieve Patient's demographics from a FHIR demographics service based on NHS Number

This call retreives basic patient demographics from the [Blackpear FHIRBall](https://github.com/BlackPearSw/fhirball) [HL7 FHIR](http://www.hl7.org/implement/standards/fhir/) service.

The root `id` element i.e. **"63436" in the JSON response example below** is the `subjectId`.

The baseURL for the HANDI-HOPD instance of the FHIRBall server is http://fhir-handihopd.rhcloud.com

i.e. the call below should be to http://fhir-handihopd.rhcloud.com/fhir/patient?identifier=7430345

#####Call: Queries the demographics store for matching parties, with the query parameters specified in the URL.
 ````
GET /fhir/patient?identifier=7430555
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
#####Returns:
````json
{
    "resourceType": "Bundle",
    "title": "Search result",
    "link": [
        {
            "rel": "self",
            "href": "http://fhir-handihopd.rhcloud.com/fhir/patient?identifier=7430345&_count=10&page=1"
        }
    ],
    "entry": [
        {
            "id": "5478959d9c903b00000a904f",
            "link": [
                {
                    "rel": "self",
                    "href": "Patient/5478959d9c903b00000a904f"
                }
            ],
            "category": [],
            "content": {
                "address": [
                    {
                        "zip": "LS23 4RT",
                        "state": "West Yorkshire",
                        "city": "Garrowhill",
                        "line": [
                            "60 Florida Gardens"
                        ],
                        "text": "60 Florida Gardens, Garrowhill, West Yorkshire, LS23 4RT"
                    }
                ],
                "birthDate": "2008-05-12T00:00:00.000Z",
                "gender": {
                    "coding": [
                        {
                            "code": "F",
                            "system": "http://hl7.org/fhir/v3/vs/AdministrativeGender"
                        }
                    ]
                },
                "name": [
                    {
                        "prefix": [
                            "Miss"
                        ],
                        "given": [
                            "Kim"
                        ],
                        "family": [
                            "Dalton"
                        ],
                        "text": "Miss Kim Dalton"
                    }
                ],
                "identifier": [
                    {
                        "value": "7430345",
                        "system": "NHS",
                        "label": "NHS"
                    }
                ],
                "resourceType": "Patient"
            }
        }
    ]
}
````
###C. Retrieve Patient's subjectId from the demographics service based on NHS Number

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

###D. Retrieve Patient's ehrId from Ehrscape based on their subjectId

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


###E. Retrieve list of Patient's recent Handover Summary compositions

Now that we have the patient's ehrId we can use it to locate their existing records.
We use an Archetype Query Language (AQL) call to retrieve a list of the identifiers and dates of existing Clinical Handover Summary encounter ``composition`` records. Compositions are document-level records which act as the container for all openEHR patient data.

#####AQL statement

````
select
    a/uid/value as uid_value,
    a/context/start_time/value as context_start_time
from EHR e[ehr_id/value='d848f3b3-25a2-4eff-bd94-acfb425cf1d8']
contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1]
where a/name/value='Clinical Handover Summary'
order by a/context/start_time/value desc
offset 0 limit 10
`````
The query API call returns a `resultset` which is a nested set of name/value pairs whose format is determined by the AQL query.

The `uid_value` element in the response is the unique identifier for the composition and `context_start_time` is the time that the document was authored.

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as uid_value, a/context/start_time/value as context_start_time from EHR e[ehr_id/value='d848f3b3-25a2-4eff-bd94-acfb425cf1d8'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1] where a/name/value='Clinical Handover Summary' order by a/context/start_time/value desc offset 0 limit
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

###F. Retrieve a single Handover Summary composition

We will use the results of the previous query to retrieve one of the compositions via its ''compositionId''.

#####Call: Returns the specified Composition in FLAT JSON format.
````
GET /rest/v1/composition/10ade55e-75d1-4278-bf3b-7ef20bcddee8::c4h_train.ehrscape.com::1?format=FLAT
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
 Content-Type: application/json
````
#####Return:
````json
{
  "composition": {
 "clinical_handover_summary/context/setting|238": true,
 "clinical_handover_summary/allergies_and_adverse_reactions/adverse_reaction:0/causative_agent|91936005": true,
 "clinical_handover_summary/current_medication/medication_statement:0/medication_item/dose_amount_description": "40mg",
 "clinical_handover_summary/plan_and_requested_actions:0/cpr_decision:0/cpr_decision|code": "at0004",
 "clinical_handover_summary/context/start_time": "2015-02-22T22:11:02.518Z",
 "clinical_handover_summary/clinical_summary:0/clinical_synopsis:0/clinical_summary": "Condition remains brittle",
 "clinical_handover_summary/history:0/reason_for_encounter:0/reason_for_admission": "Wheeze, chest tightness",
 "clinical_handover_summary/diagnoses/problem_diagnosis:0/diagnosis|code": "301485011",
 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:0/problem": "Hay fever",
 "clinical_handover_summary/plan_and_requested_actions:0/cpr_decision:0/cpr_decision|terminology": "local",
 "clinical_handover_summary/allergies_and_adverse_reactions/adverse_reaction:0/causative_agent|value": "allergy to penicillin",
 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:1/problem|code": "299757012",
 "clinical_handover_summary/diagnoses/problem_diagnosis:0/diagnosis|terminology": "SNOMED-CT",
 "clinical_handover_summary/current_medication/medication_statement:0/medication_item/medication_name|terminology": "SNOMED-CT",
 "clinical_handover_summary/allergies_and_adverse_reactions/adverse_reaction:0/causative_agent|terminology": "SNOMED-CT",
 "clinical_handover_summary/context/setting|code": "238",
 "clinical_handover_summary/admission_details/inpatient_admission:0/date_of_admission": "2015-02-17T12:30:37.553Z",
 "clinical_handover_summary/current_medication/medication_statement:0/medication_item/medication_name|320000009": true,
 "clinical_handover_summary/diagnoses/problem_diagnosis:0/date_of_onset": "2015-02-17T12:30:37.553Z",
 "clinical_handover_summary/allergies_and_adverse_reactions/adverse_reaction:0/reaction_details/date_recorded": "2012-12-17T12:30:37.553Z",
 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:1/problem|299757012": true,
 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:1/problem|terminology": "SNOMED-CT",
 "clinical_handover_summary/diagnoses/problem_diagnosis:0/diagnosis|301485011": true,
 "clinical_handover_summary/context/setting|value": "other care",
 "clinical_handover_summary/allergies_and_adverse_reactions/adverse_reaction:0/causative_agent|code": "91936005",
 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:1/date_of_onset": "2000-07-17T12:30:37.553Z",
 "clinical_handover_summary/context/setting|terminology": "openehr",
 "clinical_handover_summary/current_medication/medication_statement:0/medication_item/dose_timing_description": "at night",
 "clinical_handover_summary/plan_and_requested_actions:0/cpr_decision:0/cpr_decision|at0004": true,
 "clinical_handover_summary/plan_and_requested_actions:0/cpr_decision:0/date_of_cpr_decision": "2015-02-17T12:30:37.553Z",
 "clinical_handover_summary/diagnoses/problem_diagnosis:0/diagnosis|value": "asthma",
 "clinical_handover_summary/plan_and_requested_actions:0/cpr_decision:0/cpr_decision|value": "For attempted cardio-pulmonary resuscitation",
 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:0/date_of_onset": "1993-01-17T12:30:37.553Z",
 "clinical_handover_summary/_uid": "ee9ded3d-0b05-49ee-96c7-713631c16cee::c4h_train.ehrscape.com::1",
 "clinical_handover_summary/current_medication/medication_statement:0/medication_item/medication_name|code": "320000009",
 "clinical_handover_summary/current_medication/medication_statement:0/medication_item/medication_name|value": "Simvastatin 40mg tablet",
 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:1/problem|value": "angina pectoris"

}
````

###G. Retrieve detailed list of recent Handover Summary compositions for charting purposes

We now use a more complex Archetype Query Language (AQL) call to retrieve a the key datapoints of the most recent Handover Summary``composition``.

AQL allows for very flexible querying and output. The output chosen here is somewhat nested, in order to handle mltiple entries e.g multiple medications.

#####AQL statement
````
select
    a/uid/value as uid_value,
    a/context/start_time/value as context_start_time,
    b_a/data[at0001]/items[at0002]/value/value as Date_of_admission,
    b_b/data[at0001]/items[at0004, 'Reason for admission']/value/value as Reason_for_admission,
    b_c/data[at0001]/items[at0002, 'Clinical summary']/value as Clinical_summary,
    b_d as Medications,
    b_f as Allergies,
    b_g as Diagnoses,
    b_h as Problems,
    b_i/data[at0001]/items[at0003]/value as CPR_decision,
    b_i/data[at0001]/items[at0002]/value as Date_of_CPR_decision
from EHR e[ehr_id/value='d848f3b3-25a2-4eff-bd94-acfb425cf1d8']
contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1]
contains (
    ADMIN_ENTRY b_a[openEHR-EHR-ADMIN_ENTRY.inpatient_admission_uk.v1] or
    EVALUATION b_b[openEHR-EHR-EVALUATION.reason_for_encounter.v1] or
    EVALUATION b_c[openEHR-EHR-EVALUATION.clinical_synopsis.v1] or
    SECTION b_d[openEHR-EHR-SECTION.current_medication_rcp.v1] or
    SECTION b_f[openEHR-EHR-SECTION.allergies_adverse_reactions_rcp.v1] or
    SECTION b_g[openEHR-EHR-SECTION.diagnoses_rcp.v1] or
    SECTION b_h[openEHR-EHR-SECTION.problems_issues_rcp.v1] or
    EVALUATION b_i[openEHR-EHR-EVALUATION.cpr_decision_uk.v1])
where a/name/value='Clinical Handover Summary'
order by a/context/start_time/value desc
offset 0 limit 10

#####Call: Run AQL query and return a Resultset
````
GET /rest/v1/query?aql=select a/uid/value as uid_value, a/context/start_time/value as context_start_time, b_a/data[at0001]/items[at0002]/value/value as Date_of_admission, b_b/data[at0001]/items[at0004, 'Reason for admission']/value/value as Reason_for_admission, b_c/data[at0001]/items[at0002, 'Clinical summary']/value as Clinical_summary, b_d as Medications, b_f as Allergies, b_g as Diagnoses, b_h as Problems,  b_i/data[at0001]/items[at0003]/value as CPR_decision,  b_i/data[at0001]/items[at0002]/value as Date_of_CPR_decision from EHR e[ehr_id/value='d848f3b3-25a2-4eff-bd94-acfb425cf1d8'] contains COMPOSITION a[openEHR-EHR-COMPOSITION.report.v1] contains (  ADMIN_ENTRY b_a[openEHR-EHR-ADMIN_ENTRY.inpatient_admission_uk.v1] or     EVALUATION b_b[openEHR-EHR-EVALUATION.reason_for_encounter.v1] or  EVALUATION b_c[openEHR-EHR-EVALUATION.clinical_synopsis.v1] or SECTION b_d[openEHR-EHR-SECTION.current_medication_rcp.v1] or SECTION b_f[openEHR-EHR-SECTION.allergies_adverse_reactions_rcp.v1] or     SECTION b_g[openEHR-EHR-SECTION.diagnoses_rcp.v1] or SECTION b_h[openEHR-EHR-SECTION.problems_issues_rcp.v1] or EVALUATION b_i[openEHR-EHR-EVALUATION.cpr_decision_uk.v1]) where a/name/value='Clinical Handover Summary' order by a/context/start_time/value desc offset 0 limit 10
Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
````
#####Return:
````json
{
 "resultSet": [
        {
            "Problems": {
                "@class": "SECTION",
                "name": {
                    "@class": "DV_TEXT",
                    "value": "Problems and issues"
                },
                "archetype_details": {
                    "@class": "ARCHETYPED",
                    "archetype_id": {
                        "@class": "ARCHETYPE_ID",
                        "value": "openEHR-EHR-SECTION.problems_issues_rcp.v1"
                    },
                    "rm_version": "1.0.1"
                },
                "archetype_node_id": "openEHR-EHR-SECTION.problems_issues_rcp.v1",
                "items": [
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Problem/Diagnosis"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.problem_diagnosis.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.problem_diagnosis.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "structure"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Problem"
                                    },
                                    "archetype_node_id": "at0002",
                                    "value": {
                                        "@class": "DV_TEXT",
                                        "value": "Hay fever"
                                    }
                                },
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Date of Onset"
                                    },
                                    "archetype_node_id": "at0003",
                                    "value": {
                                        "@class": "DV_DATE_TIME",
                                        "value": "1993-01-17T13:30:37.553+01:00"
                                    }
                                }
                            ]
                        }
                    },
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Problem/Diagnosis #2"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.problem_diagnosis.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.problem_diagnosis.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "structure"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Problem"
                                    },
                                    "archetype_node_id": "at0002",
                                    "value": {
                                        "@class": "DV_CODED_TEXT",
                                        "value": "angina pectoris",
                                        "defining_code": {
                                            "@class": "CODE_PHRASE",
                                            "terminology_id": {
                                                "@class": "TERMINOLOGY_ID",
                                                "value": "SNOMED-CT"
                                            },
                                            "code_string": "299757012"
                                        }
                                    }
                                },
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Date of Onset"
                                    },
                                    "archetype_node_id": "at0003",
                                    "value": {
                                        "@class": "DV_DATE_TIME",
                                        "value": "2000-07-17T14:30:37.553+02:00"
                                    }
                                }
                            ]
                        }
                    }
                ]
            },
            "Clinical_summary": {
                "@class": "DV_TEXT",
                "value": "Worsening - started Predisolone"
            },
            "context_start_time": "2015-02-23T23:18:02.518+01:00",
            "Reason_for_admission": "Wheeze, chest tightness",
            "uid_value": "47bdd9fc-8780-4068-8465-23871934ed58::c4h_train.ehrscape.com::1",
            "Medications": {
                "@class": "SECTION",
                "name": {
                    "@class": "DV_TEXT",
                    "value": "Current medication"
                },
                "archetype_details": {
                    "@class": "ARCHETYPED",
                    "archetype_id": {
                        "@class": "ARCHETYPE_ID",
                        "value": "openEHR-EHR-SECTION.current_medication_rcp.v1"
                    },
                    "rm_version": "1.0.1"
                },
                "archetype_node_id": "openEHR-EHR-SECTION.current_medication_rcp.v1",
                "items": [
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Medication statement"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.medication_statement_uk.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.medication_statement_uk.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "Tree"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "CLUSTER",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Medication item"
                                    },
                                    "archetype_details": {
                                        "@class": "ARCHETYPED",
                                        "archetype_id": {
                                            "@class": "ARCHETYPE_ID",
                                            "value": "openEHR-EHR-CLUSTER.medication_item.v1"
                                        },
                                        "rm_version": "1.0.1"
                                    },
                                    "archetype_node_id": "openEHR-EHR-CLUSTER.medication_item.v1",
                                    "items": [
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Medication name"
                                            },
                                            "archetype_node_id": "at0001",
                                            "value": {
                                                "@class": "DV_CODED_TEXT",
                                                "value": "Salbutamol 100micrograms/dose breath actuated inhaler CFC free",
                                                "defining_code": {
                                                    "@class": "CODE_PHRASE",
                                                    "terminology_id": {
                                                        "@class": "TERMINOLOGY_ID",
                                                        "value": "SNOMED-CT"
                                                    },
                                                    "code_string": "320151000"
                                                }
                                            }
                                        },
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Dose amount description"
                                            },
                                            "archetype_node_id": "at0020",
                                            "value": {
                                                "@class": "DV_TEXT",
                                                "value": "2 puffs"
                                            }
                                        },
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Dose timing description"
                                            },
                                            "archetype_node_id": "at0021",
                                            "value": {
                                                "@class": "DV_TEXT",
                                                "value": "as required for wheeze"
                                            }
                                        }
                                    ]
                                }
                            ]
                        }
                    },
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Medication statement #2"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.medication_statement_uk.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.medication_statement_uk.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "Tree"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "CLUSTER",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Medication item"
                                    },
                                    "archetype_details": {
                                        "@class": "ARCHETYPED",
                                        "archetype_id": {
                                            "@class": "ARCHETYPE_ID",
                                            "value": "openEHR-EHR-CLUSTER.medication_item.v1"
                                        },
                                        "rm_version": "1.0.1"
                                    },
                                    "archetype_node_id": "openEHR-EHR-CLUSTER.medication_item.v1",
                                    "items": [
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Medication name"
                                            },
                                            "archetype_node_id": "at0001",
                                            "value": {
                                                "@class": "DV_CODED_TEXT",
                                                "value": "Simvastatin 40mg tablet",
                                                "defining_code": {
                                                    "@class": "CODE_PHRASE",
                                                    "terminology_id": {
                                                        "@class": "TERMINOLOGY_ID",
                                                        "value": "SNOMED-CT"
                                                    },
                                                    "code_string": "320000009"
                                                }
                                            }
                                        },
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Dose amount description"
                                            },
                                            "archetype_node_id": "at0020",
                                            "value": {
                                                "@class": "DV_TEXT",
                                                "value": "40mg"
                                            }
                                        },
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Dose timing description"
                                            },
                                            "archetype_node_id": "at0021",
                                            "value": {
                                                "@class": "DV_TEXT",
                                                "value": "at night"
                                            }
                                        }
                                    ]
                                }
                            ]
                        }
                    },
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Medication statement #3"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.medication_statement_uk.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.medication_statement_uk.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "Tree"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "CLUSTER",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Medication item"
                                    },
                                    "archetype_details": {
                                        "@class": "ARCHETYPED",
                                        "archetype_id": {
                                            "@class": "ARCHETYPE_ID",
                                            "value": "openEHR-EHR-CLUSTER.medication_item.v1"
                                        },
                                        "rm_version": "1.0.1"
                                    },
                                    "archetype_node_id": "openEHR-EHR-CLUSTER.medication_item.v1",
                                    "items": [
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Medication name"
                                            },
                                            "archetype_node_id": "at0001",
                                            "value": {
                                                "@class": "DV_CODED_TEXT",
                                                "value": "Prednisolone",
                                                "defining_code": {
                                                    "@class": "CODE_PHRASE",
                                                    "terminology_id": {
                                                        "@class": "TERMINOLOGY_ID",
                                                        "value": "SNOMED-CT"
                                                    },
                                                    "code_string": "496302014"
                                                }
                                            }
                                        },
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Dose amount description"
                                            },
                                            "archetype_node_id": "at0020",
                                            "value": {
                                                "@class": "DV_TEXT",
                                                "value": "40mg"
                                            }
                                        },
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Dose timing description"
                                            },
                                            "archetype_node_id": "at0021",
                                            "value": {
                                                "@class": "DV_TEXT",
                                                "value": "in the morning"
                                            }
                                        }
                                    ]
                                }
                            ]
                        }
                    }
                ]
            },
            "Date_of_admission": "2015-02-17T13:30:37.553+01:00",
            "Date_of_CPR_decision": {
                "@class": "DV_DATE_TIME",
                "value": "2015-02-17T13:30:37.553+01:00"
            },
            "Allergies": {
                "@class": "SECTION",
                "name": {
                    "@class": "DV_TEXT",
                    "value": "Allergies and adverse reactions"
                },
                "archetype_details": {
                    "@class": "ARCHETYPED",
                    "archetype_id": {
                        "@class": "ARCHETYPE_ID",
                        "value": "openEHR-EHR-SECTION.allergies_adverse_reactions_rcp.v1"
                    },
                    "rm_version": "1.0.1"
                },
                "archetype_node_id": "openEHR-EHR-SECTION.allergies_adverse_reactions_rcp.v1",
                "items": [
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Adverse reaction"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.adverse_reaction_uk.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.adverse_reaction_uk.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "Tree"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Causative agent"
                                    },
                                    "archetype_node_id": "at0002",
                                    "value": {
                                        "@class": "DV_CODED_TEXT",
                                        "value": "allergy to penicillin",
                                        "defining_code": {
                                            "@class": "CODE_PHRASE",
                                            "terminology_id": {
                                                "@class": "TERMINOLOGY_ID",
                                                "value": "SNOMED-CT"
                                            },
                                            "code_string": "91936005"
                                        }
                                    }
                                },
                                {
                                    "@class": "CLUSTER",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Reaction details"
                                    },
                                    "archetype_node_id": "at0025",
                                    "items": [
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Date recorded"
                                            },
                                            "archetype_node_id": "at0021",
                                            "value": {
                                                "@class": "DV_DATE_TIME",
                                                "value": "2012-12-17T13:30:37.553+01:00"
                                            }
                                        }
                                    ]
                                }
                            ]
                        }
                    }
                ]
            },
            "Diagnoses": {
                "@class": "SECTION",
                "name": {
                    "@class": "DV_TEXT",
                    "value": "Diagnoses"
                },
                "archetype_details": {
                    "@class": "ARCHETYPED",
                    "archetype_id": {
                        "@class": "ARCHETYPE_ID",
                        "value": "openEHR-EHR-SECTION.diagnoses_rcp.v1"
                    },
                    "rm_version": "1.0.1"
                },
                "archetype_node_id": "openEHR-EHR-SECTION.diagnoses_rcp.v1",
                "items": [
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Problem/Diagnosis"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.problem_diagnosis.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.problem_diagnosis.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "structure"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Diagnosis"
                                    },
                                    "archetype_node_id": "at0002",
                                    "value": {
                                        "@class": "DV_CODED_TEXT",
                                        "value": "asthma",
                                        "defining_code": {
                                            "@class": "CODE_PHRASE",
                                            "terminology_id": {
                                                "@class": "TERMINOLOGY_ID",
                                                "value": "SNOMED-CT"
                                            },
                                            "code_string": "301485011"
                                        }
                                    }
                                },
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Date of Onset"
                                    },
                                    "archetype_node_id": "at0003",
                                    "value": {
                                        "@class": "DV_DATE_TIME",
                                        "value": "2015-02-17T13:30:37.553+01:00"
                                    }
                                }
                            ]
                        }
                    }
                ]
            },
            "CPR_decision": {
                "@class": "DV_CODED_TEXT",
                "value": "For attempted cardio-pulmonary resuscitation",
                "defining_code": {
                    "@class": "CODE_PHRASE",
                    "terminology_id": {
                        "@class": "TERMINOLOGY_ID",
                        "value": "local"
                    },
                    "code_string": "at0004"
                }
            }
        },
        {
            "Problems": {
                "@class": "SECTION",
                "name": {
                    "@class": "DV_TEXT",
                    "value": "Problems and issues"
                },
                "archetype_details": {
                    "@class": "ARCHETYPED",
                    "archetype_id": {
                        "@class": "ARCHETYPE_ID",
                        "value": "openEHR-EHR-SECTION.problems_issues_rcp.v1"
                    },
                    "rm_version": "1.0.1"
                },
                "archetype_node_id": "openEHR-EHR-SECTION.problems_issues_rcp.v1",
                "items": [
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Problem/Diagnosis"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.problem_diagnosis.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.problem_diagnosis.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "structure"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Problem"
                                    },
                                    "archetype_node_id": "at0002",
                                    "value": {
                                        "@class": "DV_TEXT",
                                        "value": "Hay fever"
                                    }
                                },
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Date of Onset"
                                    },
                                    "archetype_node_id": "at0003",
                                    "value": {
                                        "@class": "DV_DATE_TIME",
                                        "value": "1993-01-17T13:30:37.553+01:00"
                                    }
                                }
                            ]
                        }
                    },
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Problem/Diagnosis #2"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.problem_diagnosis.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.problem_diagnosis.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "structure"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Problem"
                                    },
                                    "archetype_node_id": "at0002",
                                    "value": {
                                        "@class": "DV_CODED_TEXT",
                                        "value": "angina pectoris",
                                        "defining_code": {
                                            "@class": "CODE_PHRASE",
                                            "terminology_id": {
                                                "@class": "TERMINOLOGY_ID",
                                                "value": "SNOMED-CT"
                                            },
                                            "code_string": "299757012"
                                        }
                                    }
                                },
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Date of Onset"
                                    },
                                    "archetype_node_id": "at0003",
                                    "value": {
                                        "@class": "DV_DATE_TIME",
                                        "value": "2000-07-17T14:30:37.553+02:00"
                                    }
                                }
                            ]
                        }
                    }
                ]
            },
            "Clinical_summary": {
                "@class": "DV_TEXT",
                "value": "Condition remains brittle"
            },
            "context_start_time": "2015-02-22T23:11:02.518+01:00",
            "Reason_for_admission": "Wheeze, chest tightness",
            "uid_value": "f74075a7-87a4-4655-a460-03737060968e::c4h_train.ehrscape.com::1",
            "Medications": {
                "@class": "SECTION",
                "name": {
                    "@class": "DV_TEXT",
                    "value": "Current medication"
                },
                "archetype_details": {
                    "@class": "ARCHETYPED",
                    "archetype_id": {
                        "@class": "ARCHETYPE_ID",
                        "value": "openEHR-EHR-SECTION.current_medication_rcp.v1"
                    },
                    "rm_version": "1.0.1"
                },
                "archetype_node_id": "openEHR-EHR-SECTION.current_medication_rcp.v1",
                "items": [
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Medication statement"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.medication_statement_uk.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.medication_statement_uk.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "Tree"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "CLUSTER",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Medication item"
                                    },
                                    "archetype_details": {
                                        "@class": "ARCHETYPED",
                                        "archetype_id": {
                                            "@class": "ARCHETYPE_ID",
                                            "value": "openEHR-EHR-CLUSTER.medication_item.v1"
                                        },
                                        "rm_version": "1.0.1"
                                    },
                                    "archetype_node_id": "openEHR-EHR-CLUSTER.medication_item.v1",
                                    "items": [
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Medication name"
                                            },
                                            "archetype_node_id": "at0001",
                                            "value": {
                                                "@class": "DV_CODED_TEXT",
                                                "value": "Salbutamol 100micrograms/dose breath actuated inhaler CFC free",
                                                "defining_code": {
                                                    "@class": "CODE_PHRASE",
                                                    "terminology_id": {
                                                        "@class": "TERMINOLOGY_ID",
                                                        "value": "SNOMED-CT"
                                                    },
                                                    "code_string": "320151000"
                                                }
                                            }
                                        },
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Dose amount description"
                                            },
                                            "archetype_node_id": "at0020",
                                            "value": {
                                                "@class": "DV_TEXT",
                                                "value": "2 puffs"
                                            }
                                        },
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Dose timing description"
                                            },
                                            "archetype_node_id": "at0021",
                                            "value": {
                                                "@class": "DV_TEXT",
                                                "value": "as required for wheeze"
                                            }
                                        }
                                    ]
                                }
                            ]
                        }
                    },
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Medication statement #2"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.medication_statement_uk.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.medication_statement_uk.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "Tree"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "CLUSTER",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Medication item"
                                    },
                                    "archetype_details": {
                                        "@class": "ARCHETYPED",
                                        "archetype_id": {
                                            "@class": "ARCHETYPE_ID",
                                            "value": "openEHR-EHR-CLUSTER.medication_item.v1"
                                        },
                                        "rm_version": "1.0.1"
                                    },
                                    "archetype_node_id": "openEHR-EHR-CLUSTER.medication_item.v1",
                                    "items": [
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Medication name"
                                            },
                                            "archetype_node_id": "at0001",
                                            "value": {
                                                "@class": "DV_CODED_TEXT",
                                                "value": "Simvastatin 40mg tablet",
                                                "defining_code": {
                                                    "@class": "CODE_PHRASE",
                                                    "terminology_id": {
                                                        "@class": "TERMINOLOGY_ID",
                                                        "value": "SNOMED-CT"
                                                    },
                                                    "code_string": "320000009"
                                                }
                                            }
                                        },
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Dose amount description"
                                            },
                                            "archetype_node_id": "at0020",
                                            "value": {
                                                "@class": "DV_TEXT",
                                                "value": "40mg"
                                            }
                                        },
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Dose timing description"
                                            },
                                            "archetype_node_id": "at0021",
                                            "value": {
                                                "@class": "DV_TEXT",
                                                "value": "at night"
                                            }
                                        }
                                    ]
                                }
                            ]
                        }
                    }
                ]
            },
            "Date_of_admission": "2015-02-17T13:30:37.553+01:00",
            "Date_of_CPR_decision": {
                "@class": "DV_DATE_TIME",
                "value": "2015-02-17T13:30:37.553+01:00"
            },
            "Allergies": {
                "@class": "SECTION",
                "name": {
                    "@class": "DV_TEXT",
                    "value": "Allergies and adverse reactions"
                },
                "archetype_details": {
                    "@class": "ARCHETYPED",
                    "archetype_id": {
                        "@class": "ARCHETYPE_ID",
                        "value": "openEHR-EHR-SECTION.allergies_adverse_reactions_rcp.v1"
                    },
                    "rm_version": "1.0.1"
                },
                "archetype_node_id": "openEHR-EHR-SECTION.allergies_adverse_reactions_rcp.v1",
                "items": [
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Adverse reaction"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.adverse_reaction_uk.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.adverse_reaction_uk.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "Tree"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Causative agent"
                                    },
                                    "archetype_node_id": "at0002",
                                    "value": {
                                        "@class": "DV_CODED_TEXT",
                                        "value": "allergy to penicillin",
                                        "defining_code": {
                                            "@class": "CODE_PHRASE",
                                            "terminology_id": {
                                                "@class": "TERMINOLOGY_ID",
                                                "value": "SNOMED-CT"
                                            },
                                            "code_string": "91936005"
                                        }
                                    }
                                },
                                {
                                    "@class": "CLUSTER",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Reaction details"
                                    },
                                    "archetype_node_id": "at0025",
                                    "items": [
                                        {
                                            "@class": "ELEMENT",
                                            "name": {
                                                "@class": "DV_TEXT",
                                                "value": "Date recorded"
                                            },
                                            "archetype_node_id": "at0021",
                                            "value": {
                                                "@class": "DV_DATE_TIME",
                                                "value": "2012-12-17T13:30:37.553+01:00"
                                            }
                                        }
                                    ]
                                }
                            ]
                        }
                    }
                ]
            },
            "Diagnoses": {
                "@class": "SECTION",
                "name": {
                    "@class": "DV_TEXT",
                    "value": "Diagnoses"
                },
                "archetype_details": {
                    "@class": "ARCHETYPED",
                    "archetype_id": {
                        "@class": "ARCHETYPE_ID",
                        "value": "openEHR-EHR-SECTION.diagnoses_rcp.v1"
                    },
                    "rm_version": "1.0.1"
                },
                "archetype_node_id": "openEHR-EHR-SECTION.diagnoses_rcp.v1",
                "items": [
                    {
                        "@class": "EVALUATION",
                        "name": {
                            "@class": "DV_TEXT",
                            "value": "Problem/Diagnosis"
                        },
                        "archetype_details": {
                            "@class": "ARCHETYPED",
                            "archetype_id": {
                                "@class": "ARCHETYPE_ID",
                                "value": "openEHR-EHR-EVALUATION.problem_diagnosis.v1"
                            },
                            "rm_version": "1.0.1"
                        },
                        "archetype_node_id": "openEHR-EHR-EVALUATION.problem_diagnosis.v1",
                        "language": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "ISO_639-1"
                            },
                            "code_string": "en"
                        },
                        "encoding": {
                            "@class": "CODE_PHRASE",
                            "terminology_id": {
                                "@class": "TERMINOLOGY_ID",
                                "value": "IANA_character-sets"
                            },
                            "code_string": "UTF-8"
                        },
                        "subject": {
                            "@class": "PARTY_SELF"
                        },
                        "data": {
                            "@class": "ITEM_TREE",
                            "name": {
                                "@class": "DV_TEXT",
                                "value": "structure"
                            },
                            "archetype_node_id": "at0001",
                            "items": [
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Diagnosis"
                                    },
                                    "archetype_node_id": "at0002",
                                    "value": {
                                        "@class": "DV_CODED_TEXT",
                                        "value": "asthma",
                                        "defining_code": {
                                            "@class": "CODE_PHRASE",
                                            "terminology_id": {
                                                "@class": "TERMINOLOGY_ID",
                                                "value": "SNOMED-CT"
                                            },
                                            "code_string": "301485011"
                                        }
                                    }
                                },
                                {
                                    "@class": "ELEMENT",
                                    "name": {
                                        "@class": "DV_TEXT",
                                        "value": "Date of Onset"
                                    },
                                    "archetype_node_id": "at0003",
                                    "value": {
                                        "@class": "DV_DATE_TIME",
                                        "value": "2015-02-17T13:30:37.553+01:00"
                                    }
                                }
                            ]
                        }
                    }
                ]
            },
            "CPR_decision": {
                "@class": "DV_CODED_TEXT",
                "value": "For attempted cardio-pulmonary resuscitation",
                "defining_code": {
                    "@class": "CODE_PHRASE",
                    "terminology_id": {
                        "@class": "TERMINOLOGY_ID",
                        "value": "local"
                    },
                    "code_string": "at0004"
                }
            }
        }
    ]
}
````

###H. Persist a new Handover Summary Composition

All openEHR data is persisted as a COMPOSITION (document) class. openEHR data can be highly structured and potentially complex. To simplify the challenge of persisting openEHR data, examples of  'target composition' data instances have been provided in the Ehrscape ``FLAT JSON`` format.

Once the data is assembled in the correct format, the actual service call is very simple requiring only the setting of simple parameters and headers.

[Example Ehrscape Flat JSON Composition](/technical/instances/handover/Clinical handover FLAT_1.json)  

#####Call: Creates a new openEhr composition and returns the new CompositionId
````
POST /rest/v1/composition?ehrId=d848f3b3-25a2-4eff-bd94-acfb425cf1d8&templateId=C4H Asthma Diary Encounter&committerName=handi&format=FLAT

Headers:
 Ehr-Session: {{sessionId}} //The value of the sessionId
 {
   "ctx/composer_name": "Dr Joyce Smith",
   "ctx/health_care_facility|id": "999999-345",
   "ctx/health_care_facility|name": "Northumbria Community NHS",
   "ctx/id_namespace": "NHS-UK",
   "ctx/id_scheme": "2.16.840.1.113883.2.1.4.3",
   "ctx/language": "en",
   "ctx/territory": "GB",
   "ctx/time": "2015-02-23T00:11:02.518+02:00",
   "clinical_handover_summary/admission_details/inpatient_admission:0/date_of_admission": "2015-02-17T12:30:37.553Z",
   "clinical_handover_summary/clinical_summary:0/clinical_synopsis:0/clinical_summary": "Condition remains brittle",
   "clinical_handover_summary/history:0/reason_for_encounter:0/reason_for_admission": "Wheeze, chest tightness",
   "clinical_handover_summary/current_medication/medication_statement:0/medication_item/medication_name|code": "320151000",
   "clinical_handover_summary/current_medication/medication_statement:0/medication_item/medication_name|terminology": "SNOMED-CT",
   "clinical_handover_summary/current_medication/medication_statement:0/medication_item/medication_name|value": "Salbutamol 100micrograms/dose breath actuated inhaler CFC free",
   "clinical_handover_summary/current_medication/medication_statement:0/medication_item/dose_amount_description": "2 puffs",
   "clinical_handover_summary/current_medication/medication_statement:0/medication_item/dose_timing_description": "as required for wheeze",
   "clinical_handover_summary/current_medication/medication_statement:1/medication_item/medication_name|code": "320000009",
   "clinical_handover_summary/current_medication/medication_statement:1/medication_item/medication_name|terminology": "SNOMED-CT",
   "clinical_handover_summary/current_medication/medication_statement:1/medication_item/medication_name|value": "Simvastatin 40mg tablet",
   "clinical_handover_summary/current_medication/medication_statement:1/medication_item/dose_amount_description": "40mg",
   "clinical_handover_summary/current_medication/medication_statement:1/medication_item/dose_timing_description": "at night",
   "clinical_handover_summary/allergies_and_adverse_reactions/adverse_reaction:0/causative_agent|code": "91936005",
   "clinical_handover_summary/allergies_and_adverse_reactions/adverse_reaction:0/causative_agent|terminology":"SNOMED-CT",
	 "clinical_handover_summary/allergies_and_adverse_reactions/adverse_reaction:0/causative_agent|value": "allergy to penicillin",
   "clinical_handover_summary/allergies_and_adverse_reactions/adverse_reaction:0/reaction_details/date_recorded": "2012-12-17T12:30:37.553Z",
   "clinical_handover_summary/diagnoses/problem_diagnosis:0/diagnosis|code": "301485011",
   "clinical_handover_summary/diagnoses/problem_diagnosis:0/diagnosis|terminology": "SNOMED-CT",
   "clinical_handover_summary/diagnoses/problem_diagnosis:0/diagnosis|value": "asthma",
   "clinical_handover_summary/diagnoses/problem_diagnosis:0/date_of_onset": "2015-02-17T12:30:37.553Z",
 	 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:0/problem": "Hay fever",
   "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:0/date_of_onset": "1993-01-17T12:30:37.553Z",
 	 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:1/problem|code": "299757012",
 	 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:1/problem|terminology": "SNOMED-CT",
 	 "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:1/problem|value": "angina pectoris",
   "clinical_handover_summary/problems_and_issues:0/problem_diagnosis:1/date_of_onset": "2000-07-17T12:30:37.553Z",
 	 "clinical_handover_summary/plan_and_requested_actions:0/cpr_decision:0/cpr_decision|code": "at0004",
   "clinical_handover_summary/plan_and_requested_actions:0/cpr_decision:0/date_of_cpr_decision": "2015-02-17T12:30:37.553Z"
 }
````
#####Return:
````json
{
 "href": "https://rest.ehrscape.com/rest/v1/composition/8516f953-100f-4edb-8607-96120076f529::c4h_train.ehrscape.com::1"
}
````

###I. Close the Ehrscape session

The last step in working with Ehrscape is to close the session.

#####Call: Create a new openEHR session:
 ````
 DELETE /rest/v1/session?sessionId={{sessionId}}
 ````
#####Returns:
````json
{
  "sessionId": "2dcd6528-0471-4950-82fa-a018272f1339"
}
````

##J. Other Services

These are other non-Ehrscape API services.

###1. Access the ALISS 'Local community resources' service

[ALISS](http://www.aliss.org/) (A Local Information System for Scotland) is a search and collaboration tool for Health and Wellbeing resources. Originally developed in Scotland, it is now us used in regions across the UK.

Further search options e.g by locality are avaiable from the [ALISS API documentation site](http://aliss.readthedocs.org/en/latest/search_api/).

#####Call: Search the ALLIS database for resources related to asthma
````
GET http://www.aliss.org/api/v2/search/?q=asthma

Headers:
 None
````
#####Return:
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

This API call retreives NHS Choices patient information page for Asthma in HTML format.

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

The baseURL for Indizen ITSNode is http://www.itserver.es:9080/

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
