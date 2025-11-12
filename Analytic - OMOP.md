# FHIR Analytics Pipeline with OMOP CDM Integration

## Table of Contents
1. [Overview](#overview)
2. [Objectives](#objectives)
3. [Architecture & Design](#architecture--design)
4. [FHIR to OMOP Mapping](#fhir-to-omop-mapping)
5. [Implementation Guide](#implementation-guide)
6. [Challenges & Solutions](#challenges--solutions)
7. [Best Practices](#best-practices)
8. [References](#references)

---

## Overview

This document outlines the design and implementation of a comprehensive analytics pipeline that transforms FHIR (Fast Healthcare Interoperability Resources) data into the OMOP Common Data Model for research and analytics purposes. The pipeline leverages open-source tools and modern data engineering practices to handle complex healthcare data transformations at scale.

### Why FHIR + OMOP?

**FHIR Strengths:**
- Real-time clinical data exchange
- Workflow-oriented design
- Interoperability-focused
- Rich contextual information
- Modern REST API architecture

**OMOP CDM Strengths:**
- Standardized for retrospective research
- Optimized for observational studies
- Common vocabulary across institutions
- Supports multi-site collaborative research
- Proven analytics capabilities

**Integration Benefits:**
- Combines operational and analytical capabilities
- Preserves clinical context while enabling research
- Enables both real-time and batch analytics
- Facilitates EHR-to-research workflows

---

## Objectives

### Primary Goals
1. **Build Automated Analytics Pipeline**
   - Create simple, automated, batch analytics pipeline for FHIR data
   - Utilize open-source software (Apache Spark, Pathling)
   - Enable reproducible, scalable data transformations

2. **Transform Complex FHIR to Tabular Views**
   - Use FHIRPath and SQL to transform nested FHIR data
   - Generate use-case centric tabular views
   - Maintain semantic accuracy during transformation

3. **Handle FHIR Complexities**
   - Codes and terminology (SNOMED CT, LOINC, RxNorm)
   - Questionnaire responses and observations
   - FHIR extensions and custom profiles
   - Nested and repeating elements
   - Status fields and data completeness

4. **Enable OMOP CDM Integration**
   - Map FHIR resources to OMOP domains
   - Handle vocabulary standardization
   - Preserve data lineage and provenance
   - Support both FHIR and OMOP analytics workflows

### Success Criteria
- < 1% data validation errors post-transformation
- Support for 10+ core FHIR resource types
- Processing capability for millions of records
- Terminology mapping accuracy > 95%
- Automated pipeline execution with monitoring

---

## Architecture & Design

### System Architecture

![](https://i.imgur.com/1HbS1nM.png)

**Component Overview:**

1. **Data Ingestion Layer**
   - FHIR Server (source)
   - Bulk Data Export (FHIR Bulk Data Access IG)
   - NDJSON file staging

2. **Processing Layer**
   - Apache Spark cluster
   - Pathling FHIR analytics engine
   - FHIRPath query execution
   - Terminology services (Ontoserver)

3. **Transformation Layer**
   - FHIR to tabular transformation
   - OMOP CDM mapping
   - Vocabulary standardization
   - Data quality checks

4. **Storage Layer**
   - Parquet/Delta Lake format
   - Partitioned by resource type
   - Versioned datasets

5. **Analytics Layer**
   - SQL analytics
   - Reporting dashboards
   - Research queries

### Data Flow

![](https://i.imgur.com/LshlPU0.png)

**Pipeline Stages:**

1. **Extract**: Bulk export from FHIR server → NDJSON files
2. **Load**: NDJSON → Spark DataFrames via Pathling encoders
3. **Transform**:
   - FHIRPath queries for data extraction
   - SQL transformations for reshaping
   - Terminology lookups and standardization
4. **Map**: FHIR resources → OMOP CDM tables
5. **Validate**: Data quality checks and validation rules
6. **Store**: Write to Parquet/Delta format
7. **Analyze**: SQL queries and analytics

---

## FHIR to OMOP Mapping

### Mapping Overview

![](https://i.imgur.com/R2wsjJF.png)

### Core Resource Mappings

#### Patient → PERSON Table
```sql
-- FHIR Patient to OMOP PERSON
SELECT
  patient.id AS person_id,
  CASE patient.gender
    WHEN 'male' THEN 8507
    WHEN 'female' THEN 8532
  END AS gender_concept_id,
  YEAR(patient.birthDate) AS year_of_birth,
  MONTH(patient.birthDate) AS month_of_birth,
  DAY(patient.birthDate) AS day_of_birth,
  -- Race and ethnicity from extensions
  get_race_concept(patient.extension) AS race_concept_id,
  get_ethnicity_concept(patient.extension) AS ethnicity_concept_id
FROM patient_dataset
```

#### Observation → OBSERVATION Table
```sql
-- FHIR Observation to OMOP OBSERVATION
SELECT
  observation.subject.reference AS person_id,
  get_standard_concept(observation.code) AS observation_concept_id,
  observation.effectiveDateTime AS observation_date,
  observation.valueQuantity.value AS value_as_number,
  observation.valueQuantity.unit AS unit_source_value,
  get_concept_id(observation.valueCodeableConcept) AS value_as_concept_id
FROM observation_dataset
WHERE observation.status = 'final' -- Only completed observations
```

#### MedicationRequest → DRUG_EXPOSURE Table
```sql
-- FHIR MedicationRequest to OMOP DRUG_EXPOSURE
SELECT
  medicationRequest.subject.reference AS person_id,
  get_rxnorm_concept(medicationRequest.medicationCodeableConcept) AS drug_concept_id,
  medicationRequest.authoredOn AS drug_exposure_start_date,
  COALESCE(
    medicationRequest.dosageInstruction[0].timing.repeat.boundsPeriod.end,
    DATE_ADD(medicationRequest.authoredOn,
             medicationRequest.dispenseRequest.expectedSupplyDuration.value)
  ) AS drug_exposure_end_date,
  medicationRequest.dosageInstruction[0].doseAndRate[0].doseQuantity.value AS quantity
FROM medicationrequest_dataset
WHERE medicationRequest.status IN ('active', 'completed')
```

#### Condition → CONDITION_OCCURRENCE Table
```sql
-- FHIR Condition to OMOP CONDITION_OCCURRENCE
SELECT
  condition.subject.reference AS person_id,
  get_snomed_concept(condition.code) AS condition_concept_id,
  COALESCE(condition.onsetDateTime, condition.recordedDate) AS condition_start_date,
  condition.abatementDateTime AS condition_end_date,
  CASE condition.verificationStatus.coding[0].code
    WHEN 'confirmed' THEN 32902 -- EHR
    WHEN 'provisional' THEN 32901 -- Preliminary
  END AS condition_type_concept_id
FROM condition_dataset
WHERE condition.clinicalStatus.coding[0].code = 'active'
```

### OMOP Domain Mapping

| FHIR Resource | OMOP Domain | Primary Table | Notes |
|--------------|-------------|---------------|-------|
| Patient | Person | PERSON | Core demographics |
| Observation | Observation | OBSERVATION | Labs, vitals, assessments |
| Condition | Condition | CONDITION_OCCURRENCE | Diagnoses, problems |
| MedicationRequest | Drug | DRUG_EXPOSURE | Prescriptions |
| MedicationAdministration | Drug | DRUG_EXPOSURE | Actual administrations |
| Procedure | Procedure | PROCEDURE_OCCURRENCE | Performed procedures |
| Encounter | Visit | VISIT_OCCURRENCE | Healthcare visits |
| AllergyIntolerance | Observation | OBSERVATION | Allergies as observations |
| DiagnosticReport | Measurement | MEASUREMENT | Diagnostic results |
| Immunization | Drug | DRUG_EXPOSURE | Vaccines as drugs |
| Device | Device | DEVICE_EXPOSURE | Medical devices |

### Vocabulary Mapping Strategy

**Terminology Standardization:**

| Source System | Target OMOP Vocabulary | Mapping Method |
|--------------|----------------------|----------------|
| SNOMED CT | SNOMED | Direct mapping |
| LOINC | LOINC | Direct mapping |
| RxNorm | RxNorm | Direct mapping |
| ICD-10 | SNOMED | USAGI + manual review |
| CPT | SNOMED | USAGI + manual review |
| Local codes | Custom concepts | 2-billion range |

**Mapping Process:**
1. **Direct Mapping**: Use OMOP vocabulary tables for standard codes
2. **Terminology Server**: Query Ontoserver for equivalence mappings
3. **USAGI Tool**: Semi-automated mapping for non-standard codes
4. **Custom Concepts**: Create 2-billion+ concept_ids for unmapped codes
5. **Manual Review**: Expert review for high-impact mappings
---

## Implementation Guide

### Technology Stack

#### Core Technologies
| Component | Technology | Purpose |
|-----------|-----------|---------|
| Processing Engine | Apache Spark 3.5+ | Distributed data processing |
| FHIR Analytics | Pathling 6.x | FHIR resource parsing and FHIRPath queries |
| Terminology Service | Ontoserver | FHIR terminology operations (CodeSystem, ValueSet, ConceptMap) |
| Languages | Python (PySpark), SQL, FHIRPath | Data transformation and query |
| Data Formats | NDJSON (input), Parquet/Delta (storage) | Serialization and storage |
| Orchestration | Apache Airflow / Databricks Workflows | Pipeline scheduling and monitoring |

#### Development Environment
| Component | Technology | Purpose |
|-----------|-----------|---------|
| Platform | Databricks / Local Spark | Development and execution environment |
| Notebooks | Jupyter / Databricks Notebooks | Exploratory analysis and prototyping |
| Version Control | Git | Code versioning |
| Testing | pytest, Great Expectations | Unit tests and data quality validation |

### Setup Instructions

#### 1. Environment Setup

**Install Python Dependencies:**
```bash
pip install pathling pyspark>=3.5.0 delta-spark great-expectations
```

**Configure Spark:**
```python
from pyspark.sql import SparkSession
from pathling import PathlingContext

# Initialize Spark with Pathling
spark = SparkSession.builder \
    .appName("FHIR-OMOP-Pipeline") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .config("spark.jars.packages", "au.csiro.pathling:pathling-spark:6.4.2") \
    .getOrCreate()

# Initialize Pathling context
pc = PathlingContext.create(spark)
```

**Configure Ontoserver Connection:**
```python
# Set terminology server endpoint
pc.configure({
    'terminology.serverUrl': 'https://ontoserver.example.com/fhir',
    'terminology.authentication.enabled': True,
    'terminology.authentication.tokenEndpoint': 'https://auth.example.com/token'
})
```

#### 2. Data Ingestion

**Extract FHIR Data via Bulk Export:**
```python
import requests

# FHIR Bulk Data Export (using SMART Backend Services)
bulk_export_url = "https://fhir-server.example.com/$export"
headers = {
    "Authorization": f"Bearer {access_token}",
    "Accept": "application/fhir+json",
    "Prefer": "respond-async"
}

# Initiate export
response = requests.get(bulk_export_url, headers=headers)
status_url = response.headers['Content-Location']

# Poll for completion and download NDJSON files
# ... (implementation details)
```

**Load FHIR Data with Pathling:**
```python
# Read NDJSON files into Pathling datasets
patient_df = pc.read.ndjson('/data/fhir/Patient.ndjson')
observation_df = pc.read.ndjson('/data/fhir/Observation.ndjson')
condition_df = pc.read.ndjson('/data/fhir/Condition.ndjson')
medication_df = pc.read.ndjson('/data/fhir/MedicationRequest.ndjson')
procedure_df = pc.read.ndjson('/data/fhir/Procedure.ndjson')
encounter_df = pc.read.ndjson('/data/fhir/Encounter.ndjson')

# Register as SQL views
patient_df.createOrReplaceTempView('patient')
observation_df.createOrReplaceTempView('observation')
condition_df.createOrReplaceTempView('condition')
```

#### 3. FHIRPath Transformations

**Example: Extract Patient Demographics**
```python
# Use FHIRPath to extract and flatten patient data
patient_demographics = patient_df.select(
    pc.extract_literal(patient_df, "id") as "patient_id",
    pc.extract_literal(patient_df, "birthDate") as "birth_date",
    pc.extract_literal(patient_df, "gender") as "gender",
    pc.extract_literal(patient_df, "address.first().postalCode") as "zip_code",
    pc.extract_literal(patient_df, "extension.where(url='http://hl7.org/fhir/us/core/StructureDefinition/us-core-race').extension.first().valueCoding.code") as "race_code"
)
```

**Example: Filter Observations by Code**
```python
# Get all blood pressure observations using terminology
bp_observations = observation_df.filter(
    pc.member_of(
        observation_df,
        "code",
        "http://hl7.org/fhir/ValueSet/observation-vitalsignresult"
    )
)
```

#### 4. OMOP Transformation Examples

**Create PERSON Table:**
```python
from pyspark.sql.functions import year, month, dayofmonth, when, col

person_omop = spark.sql("""
SELECT
    CAST(p.id AS BIGINT) AS person_id,
    CASE p.gender
        WHEN 'male' THEN 8507
        WHEN 'female' THEN 8532
        WHEN 'other' THEN 8521
        ELSE 0
    END AS gender_concept_id,
    YEAR(p.birthDate) AS year_of_birth,
    MONTH(p.birthDate) AS month_of_birth,
    DAY(p.birthDate) AS day_of_birth,
    p.birthDate AS birth_datetime,
    -- Extract race from US Core extension
    COALESCE(
        get_extension_value(p.extension, 'us-core-race'),
        0
    ) AS race_concept_id,
    -- Extract ethnicity from US Core extension
    COALESCE(
        get_extension_value(p.extension, 'us-core-ethnicity'),
        0
    ) AS ethnicity_concept_id,
    -- Location from address
    get_location_id(p.address[0].postalCode) AS location_id,
    -- Provider and care site
    NULL AS provider_id,
    NULL AS care_site_id,
    -- Source values
    p.id AS person_source_value,
    p.gender AS gender_source_value
FROM patient p
""")

# Write to Delta table
person_omop.write.format("delta").mode("overwrite").save("/omop/person")
```

**Create OBSERVATION Table:**
```python
observation_omop = spark.sql("""
WITH obs_flattened AS (
    SELECT
        CAST(REPLACE(o.subject.reference, 'Patient/', '') AS BIGINT) AS person_id,
        o.id AS observation_id,
        o.code.coding[0] AS code,
        o.effectiveDateTime AS effective_datetime,
        o.valueQuantity.value AS value_number,
        o.valueQuantity.unit AS unit,
        o.valueCodeableConcept.coding[0] AS value_code,
        o.status
    FROM observation o
    WHERE o.status = 'final'
)
SELECT
    observation_id AS observation_id,
    person_id,
    map_to_standard_concept(code.system, code.code) AS observation_concept_id,
    DATE(effective_datetime) AS observation_date,
    effective_datetime AS observation_datetime,
    32817 AS observation_type_concept_id, -- EHR
    value_number AS value_as_number,
    map_to_standard_concept(value_code.system, value_code.code) AS value_as_concept_id,
    NULL AS qualifier_concept_id,
    NULL AS unit_concept_id,
    NULL AS provider_id,
    NULL AS visit_occurrence_id,
    NULL AS visit_detail_id,
    code.code AS observation_source_value,
    code.system AS observation_source_concept_id,
    unit AS unit_source_value,
    value_code.display AS qualifier_source_value
FROM obs_flattened
""")

observation_omop.write.format("delta").mode("overwrite").save("/omop/observation")
```

#### 5. Terminology Mapping Functions

**Create UDF for Concept Mapping:**
```python
from pyspark.sql.types import IntegerType
import requests

# Concept mapping cache
concept_cache = {}

def map_to_standard_concept(system, code):
    """
    Map source code to OMOP standard concept_id using terminology server
    """
    cache_key = f"{system}|{code}"

    if cache_key in concept_cache:
        return concept_cache[cache_key]

    # Query Ontoserver for mapping
    params = {
        'system': system,
        'code': code,
        'target': 'http://ohdsi.org/omop/vocab' # OMOP vocabulary
    }

    response = requests.get(
        'https://ontoserver.example.com/fhir/ConceptMap/$translate',
        params=params
    )

    if response.status_code == 200:
        result = response.json()
        if result.get('parameter', []):
            concept_id = result['parameter'][0].get('valueInteger', 0)
            concept_cache[cache_key] = concept_id
            return concept_id

    # Return 0 for unmapped codes
    return 0

# Register as Spark UDF
spark.udf.register("map_to_standard_concept", map_to_standard_concept, IntegerType())
```

#### 6. Data Quality Validation

**Implement Great Expectations Checks:**
```python
import great_expectations as ge

# Convert to Great Expectations DataFrame
person_ge = ge.from_pandas(person_omop.toPandas())

# Define expectations
person_ge.expect_column_values_to_not_be_null('person_id')
person_ge.expect_column_values_to_be_unique('person_id')
person_ge.expect_column_values_to_be_between('year_of_birth', 1900, 2025)
person_ge.expect_column_values_to_be_in_set('gender_concept_id', [8507, 8532, 8521, 0])

# Validate
validation_results = person_ge.validate()

if not validation_results.success:
    print("Validation failed!")
    print(validation_results)
else:
    print("All validations passed!")
```

#### 7. Pipeline Orchestration

**Databricks Workflow Example:**
```python
# Databricks notebook: 01_extract_fhir.py
# Task 1: Extract FHIR data from source
dbutils.notebook.run("extract_bulk_fhir", timeout_seconds=3600)

# Databricks notebook: 02_transform_omop.py
# Task 2: Transform to OMOP
dbutils.notebook.run("transform_to_omop", timeout_seconds=7200)

# Databricks notebook: 03_validate.py
# Task 3: Validate data quality
dbutils.notebook.run("validate_omop", timeout_seconds=1800)
```

**Apache Airflow DAG:**
```python
from airflow import DAG
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'start_date': datetime(2025, 1, 1),
    'retries': 2,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'fhir_omop_pipeline',
    default_args=default_args,
    schedule_interval='@daily',
    catchup=False
)

extract_task = SparkSubmitOperator(
    task_id='extract_fhir',
    application='/jobs/extract_fhir.py',
    dag=dag
)

transform_task = SparkSubmitOperator(
    task_id='transform_omop',
    application='/jobs/transform_omop.py',
    dag=dag
)

validate_task = SparkSubmitOperator(
    task_id='validate_quality',
    application='/jobs/validate_omop.py',
    dag=dag
)

extract_task >> transform_task >> validate_task
```

### Sample Data Sources

**HL7 Sample Data:**
- Synthea: https://github.com/synthetichealth/synthea
- HL7 FHIR Examples: http://hl7.org/fhir/examples.html
- SMART on FHIR Test Data: https://docs.smarthealthit.org/

**OMOP Vocabularies:**
- Athena: https://athena.ohdsi.org/
- Download standard vocabularies (SNOMED, LOINC, RxNorm, etc.)

---

## Challenges & Solutions

### 1. Structural and Philosophical Differences

**Challenge:**
- FHIR is designed for real-time clinical data exchange with a workflow-oriented approach
- OMOP is structured for retrospective research and observational analysis
- Direct mappings can misrepresent data due to these fundamental differences

**Solutions:**
- **Context Preservation**: Use OMOP's `_source_value` and `_source_concept_id` fields to retain original FHIR context
- **Observation Domain Fallback**: Map contextually rich FHIR data that doesn't fit standard OMOP domains to the OBSERVATION table
- **Custom Metadata**: Store FHIR-specific context (e.g., `reasonCode`, `category`) in the NOTE table or custom fields
- **Bi-directional Mapping**: Maintain FHIR references alongside OMOP data for traceability

**Example:**
```python
# Preserve FHIR context in OMOP
procedure_with_context = spark.sql("""
SELECT
    p.id AS procedure_occurrence_id,
    p.subject.reference AS person_id,
    map_concept(p.code) AS procedure_concept_id,
    -- Preserve reason for procedure
    map_concept(p.reasonCode[0]) AS modifier_concept_id,
    -- Store full FHIR resource as JSON in note
    to_json(p) AS note_text
FROM procedure p
""")
```

### 2. Vocabulary and Terminology Mapping

**Challenge:**
- High unmapping rates for institution-specific codes (Epic: 45.8% encounters, 51.6% procedures, 93.5% devices)
- Legacy coding systems deprecated from current OMOP vocabularies
- One-to-many relationships between source codes and OMOP concepts
- Loss of granularity when mapping to standard concepts

**Solutions:**

**a) Custom Concepts ("2-Billionaires"):**
```python
def create_custom_concept(source_code, source_system, domain):
    """
    Create custom OMOP concept for unmapped codes
    Concept IDs start at 2,000,000,000
    """
    custom_concept_id = 2_000_000_000 + hash(f"{source_system}|{source_code}") % 1_000_000_000

    return {
        'concept_id': custom_concept_id,
        'concept_name': f"Custom: {source_code}",
        'domain_id': domain,
        'vocabulary_id': source_system,
        'concept_class_id': 'Custom',
        'standard_concept': None,  # Not a standard concept
        'concept_code': source_code,
        'valid_start_date': '2025-01-01',
        'valid_end_date': '2099-12-31',
        'invalid_reason': None
    }
```

**b) Multi-step Mapping Strategy:**
```python
def map_code_with_fallback(system, code):
    """
    Multi-level mapping with fallbacks
    """
    # Level 1: Direct OMOP vocabulary lookup
    concept_id = omop_vocab_lookup(system, code)
    if concept_id:
        return concept_id, 'direct'

    # Level 2: Terminology server equivalence
    concept_id = terminology_server_translate(system, code)
    if concept_id:
        return concept_id, 'equivalent'

    # Level 3: Parent/ancestor mapping
    concept_id = map_to_parent_concept(system, code)
    if concept_id:
        return concept_id, 'parent'

    # Level 4: Custom concept creation
    concept_id = create_custom_concept(code, system, infer_domain(code))
    return concept_id, 'custom'
```

**c) USAGI-based Mapping:**
```bash
# Use USAGI for semi-automated source-to-concept mapping
# 1. Export unmapped codes
# 2. Import to USAGI
# 3. Review suggested mappings
# 4. Approve and export mapping table
```

### 3. Data Status and Completeness

**Challenge:**
- FHIR resources have status fields (planned, in-progress, completed, cancelled)
- OMOP typically expects only completed/confirmed activities
- MedicationRequest represents intent, not actual exposure
- Different verificationStatus values affect data reliability

**Solutions:**

**a) Status Filtering:**
```sql
-- Only include completed observations
SELECT * FROM observation
WHERE observation.status IN ('final', 'amended', 'corrected')

-- Exclude entered-in-error
AND observation.status != 'entered-in-error'

-- Handle verification status for conditions
SELECT * FROM condition
WHERE condition.verificationStatus.coding[0].code IN ('confirmed', 'provisional')
AND condition.clinicalStatus.coding[0].code != 'inactive'
```

**b) Type Concept Differentiation:**
```python
# Use different type_concept_id values to indicate data quality
status_to_type_concept = {
    'final': 32817,           # EHR
    'preliminary': 32901,      # Preliminary
    'amended': 32902,          # EHR with validation
    'confirmed': 32817,        # EHR
    'provisional': 32890       # Patient reported
}
```

**c) MedicationRequest vs MedicationAdministration:**
```sql
-- Separate prescriptions from administrations
CREATE VIEW drug_orders AS
SELECT *, 38000177 AS drug_type_concept_id  -- Prescription written
FROM medicationrequest
WHERE status IN ('active', 'completed');

CREATE VIEW drug_administrations AS
SELECT *, 38000180 AS drug_type_concept_id  -- Inpatient administration
FROM medicationadministration
WHERE status = 'completed';
```

### 4. Nested and Repeating Elements

**Challenge:**
- FHIR has deeply nested structures (extensions, coding arrays, components)
- OMOP has flat, tabular structure
- Multiple values in arrays need to be exploded
- Risk of data loss or misrepresentation

**Solutions:**

**a) Array Explosion:**
```python
from pyspark.sql.functions import explode, col

# Handle multiple codings
observation_exploded = observation_df.select(
    col("id"),
    col("subject.reference").alias("patient_ref"),
    explode(col("code.coding")).alias("coding")
).select(
    "id",
    "patient_ref",
    "coding.system",
    "coding.code",
    "coding.display"
)
```

**b) Component Handling (Blood Pressure):**
```sql
-- Split multi-component observations
WITH bp_components AS (
    SELECT
        o.id,
        o.subject.reference AS person_id,
        o.effectiveDateTime,
        c.code.coding[0].code AS component_code,
        c.valueQuantity.value AS component_value
    FROM observation o
    LATERAL VIEW explode(o.component) AS c
    WHERE exists(
        SELECT 1 FROM explode(o.code.coding) AS coding
        WHERE coding.code = '85354-9' -- Blood pressure panel
    )
)
SELECT
    person_id,
    effectiveDateTime AS observation_date,
    MAX(CASE WHEN component_code = '8480-6' THEN component_value END) AS systolic_bp,
    MAX(CASE WHEN component_code = '8462-4' THEN component_value END) AS diastolic_bp
FROM bp_components
GROUP BY person_id, effectiveDateTime
```

**c) Extension Extraction:**
```python
def extract_extension(extensions, url):
    """
    Extract value from FHIR extension by URL
    """
    if not extensions:
        return None

    for ext in extensions:
        if ext.get('url') == url:
            # Handle different value types
            for key in ext.keys():
                if key.startswith('value'):
                    return ext[key]
    return None

# Use in Spark SQL
spark.udf.register("extract_extension", extract_extension)
```

### 5. One-to-Many Relationships

**Challenge:**
- A single FHIR resource may map to multiple OMOP tables
- A single coded value may map to multiple OMOP concepts or domains
- Encounter resources contain visit, provider, and location information

**Solutions:**

**a) Fragment Processing:**
```python
# Break FHIR Encounter into multiple OMOP tables
def process_encounter_fragments(encounter_df):
    """
    Process Encounter into VISIT_OCCURRENCE, PROVIDER, CARE_SITE
    """

    # Fragment 1: Visit occurrence
    visit_occurrence = encounter_df.select(
        col("id").alias("visit_occurrence_id"),
        col("subject.reference").alias("person_id"),
        col("class.code").alias("visit_concept_id"),
        col("period.start").alias("visit_start_datetime"),
        col("period.end").alias("visit_end_datetime")
    )

    # Fragment 2: Provider linkage
    encounter_provider = encounter_df.select(
        col("id").alias("visit_occurrence_id"),
        explode(col("participant")).alias("participant")
    ).select(
        "visit_occurrence_id",
        col("participant.individual.reference").alias("provider_id")
    )

    # Fragment 3: Care site
    encounter_location = encounter_df.select(
        col("id").alias("visit_occurrence_id"),
        explode(col("location")).alias("location")
    ).select(
        "visit_occurrence_id",
        col("location.location.reference").alias("care_site_id")
    )

    return visit_occurrence, encounter_provider, encounter_location
```

**b) Multi-domain Mapping:**
```python
# Procedure can map to both PROCEDURE_OCCURRENCE and DEVICE_EXPOSURE
procedures_df = spark.sql("""
SELECT
    p.id,
    p.subject.reference AS person_id,
    p.performedDateTime AS procedure_date,
    p.code AS procedure_code,
    device.code AS device_code
FROM procedure p
LATERAL VIEW explode(p.focalDevice) AS device
""")

# Split into separate tables
procedure_occurrence = procedures_df.select(
    "id", "person_id", "procedure_date", "procedure_code"
)

device_exposure = procedures_df.where(col("device_code").isNotNull()).select(
    "id", "person_id", "procedure_date", "device_code"
)
```

### 6. Performance and Scalability

**Challenge:**
- Large FHIR datasets (millions of resources)
- Complex transformations and terminology lookups
- Real-time vs batch processing trade-offs

**Solutions:**

**a) Partitioning Strategy:**
```python
# Partition by resource type and date
observation_omop.write \
    .format("delta") \
    .partitionBy("observation_date") \
    .mode("overwrite") \
    .save("/omop/observation")
```

**b) Caching and Broadcast:**
```python
# Cache frequently accessed vocabulary tables
vocab_df = spark.read.parquet("/omop/concept")
vocab_df.cache()
spark.sparkContext.broadcast(vocab_df)

# Use broadcast join for small lookup tables
result = large_df.join(
    broadcast(vocab_df),
    large_df.source_code == vocab_df.concept_code
)
```

**c) Incremental Processing:**
```python
# Process only new/updated resources
last_run_timestamp = get_last_run_timestamp()

new_observations = spark.read.ndjson("/fhir/Observation*.ndjson") \
    .filter(col("meta.lastUpdated") > last_run_timestamp)

# Upsert to Delta table
new_observations.write \
    .format("delta") \
    .mode("merge") \
    .save("/omop/observation")
```

---

## Best Practices

### 1. Data Quality and Validation

**Always Validate:**
```python
# Define comprehensive data quality checks
validation_rules = {
    'person': [
        'person_id is unique',
        'year_of_birth between 1900 and 2025',
        'gender_concept_id in valid set',
        'person_source_value is not null'
    ],
    'observation': [
        'person_id exists in person table',
        'observation_date <= current_date',
        'observation_concept_id != 0',
        'value_as_number within valid range for concept'
    ]
}

# Implement referential integrity checks
spark.sql("""
SELECT COUNT(*) AS orphaned_observations
FROM observation o
LEFT JOIN person p ON o.person_id = p.person_id
WHERE p.person_id IS NULL
""")
```

**Data Quality Metrics:**
- Mapping success rate (target: > 95%)
- Null rate by field (target: < 5% for required fields)
- Referential integrity violations (target: 0)
- Duplicate records (target: 0)
- Date validity (target: 100%)

### 2. Documentation and Lineage

**Document Everything:**
- Mapping decisions and assumptions
- Custom concept definitions
- Transformation logic
- Data quality rules
- Known limitations

**Maintain Lineage:**
```python
# Store provenance metadata
provenance = {
    'source_fhir_resource': 'Observation/12345',
    'source_fhir_server': 'https://fhir.example.com',
    'transformation_version': 'v2.1.0',
    'transformation_timestamp': '2025-01-12T10:30:00Z',
    'mapper': 'fhir_omop_pipeline',
    'validation_status': 'passed'
}
```

### 3. Incremental Development

**Start Small:**
1. Begin with core resources (Patient, Observation, Condition)
2. Validate thoroughly before expanding
3. Add complexity incrementally
4. Test with synthetic data first (Synthea)

**Iterative Approach:**
- Week 1: Patient → PERSON
- Week 2: Observation → OBSERVATION
- Week 3: Condition → CONDITION_OCCURRENCE
- Week 4: MedicationRequest → DRUG_EXPOSURE
- Week 5: Integration and validation

### 4. Terminology Management

**Best Practices:**
- Download latest OMOP vocabularies from Athena quarterly
- Maintain local vocabulary cache for performance
- Document custom concept_ids in a registry
- Review unmapped codes monthly
- Engage domain experts for critical mappings

**Vocabulary Updates:**
```python
# Schedule regular vocabulary refresh
def refresh_vocabularies():
    # 1. Download from Athena
    download_athena_vocabularies()

    # 2. Load to Spark
    new_vocab = spark.read.csv("/vocabs/new/CONCEPT.csv")

    # 3. Compare with existing
    changes = compare_vocabularies(current_vocab, new_vocab)

    # 4. Update and remaps
    if changes:
        remap_affected_records(changes)
        update_vocabulary_tables(new_vocab)
```

### 5. Performance Optimization

**Optimization Checklist:**
- ✓ Use Delta Lake for ACID transactions and time travel
- ✓ Partition large tables by date
- ✓ Cache vocabulary tables
- ✓ Use broadcast joins for small tables
- ✓ Minimize shuffles in transformations
- ✓ Use column pruning and predicate pushdown
- ✓ Monitor and tune Spark configuration

**Spark Configuration:**
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.files.maxPartitionBytes", "128MB")
spark.conf.set("spark.sql.shuffle.partitions", "200")
```

### 6. Testing Strategy

**Multi-level Testing:**

**Unit Tests:**
```python
def test_patient_to_person_mapping():
    # Arrange
    sample_patient = create_sample_patient()

    # Act
    person = transform_patient_to_person(sample_patient)

    # Assert
    assert person['person_id'] == sample_patient['id']
    assert person['gender_concept_id'] in [8507, 8532, 8521]
    assert person['year_of_birth'] == 1980
```

**Integration Tests:**
```python
def test_end_to_end_pipeline():
    # Load test FHIR data
    test_data = load_synthea_sample()

    # Run pipeline
    result = run_pipeline(test_data)

    # Validate OMOP output
    assert result['person'].count() > 0
    assert result['observation'].count() > 0
    assert no_referential_integrity_violations(result)
```

**Data Quality Tests:**
```python
# Great Expectations suite
expectations = [
    "expect_column_values_to_be_unique('person_id')",
    "expect_column_values_to_not_be_null('birth_date')",
    "expect_column_min_to_be_between('year_of_birth', 1900, 2025)"
]
```

### 7. Monitoring and Observability

**Pipeline Monitoring:**
```python
# Log key metrics
metrics = {
    'records_processed': len(input_df),
    'records_mapped': len(output_df),
    'mapping_success_rate': len(output_df) / len(input_df),
    'unmapped_codes': count_unmapped_codes(),
    'validation_errors': count_validation_errors(),
    'processing_time_seconds': elapsed_time
}

log_to_monitoring_system(metrics)
```

**Alerting:**
- Alert if mapping success rate < 90%
- Alert if validation error rate > 1%
- Alert if processing time > 2x historical average
- Alert on pipeline failures

### 8. Security and Privacy

**PHI Protection:**
- De-identify data according to HIPAA Safe Harbor or Expert Determination
- Implement row-level security in analytics layer
- Audit all data access
- Encrypt data at rest and in transit
- Use secure terminology server connections

**De-identification Example:**
```python
# Hash identifiers
from hashlib import sha256

def hash_identifier(value, salt):
    return sha256(f"{value}{salt}".encode()).hexdigest()[:16]

# Remove dates (shift or remove)
def shift_date(date, shift_days):
    return date + timedelta(days=shift_days)

# Generalize locations
def generalize_zipcode(zipcode):
    return zipcode[:3] + "00"  # 3-digit ZIP
```

---

## References

### Official Documentation

**FHIR Standards:**
- HL7 FHIR Specification: http://hl7.org/fhir/
- FHIR to OMOP Implementation Guide: https://build.fhir.org/ig/HL7/fhir-omop-ig/
- FHIR Bulk Data Access IG: https://hl7.org/fhir/uv/bulkdata/
- US Core Implementation Guide: http://hl7.org/fhir/us/core/

**OMOP CDM:**
- OMOP CDM Documentation: https://ohdsi.github.io/CommonDataModel/
- OHDSI Book: https://ohdsi.github.io/TheBookOfOhdsi/
- Athena Vocabulary: https://athena.ohdsi.org/
- OHDSI Forums: https://forums.ohdsi.org/

**Technologies:**
- Apache Spark Documentation: https://spark.apache.org/docs/latest/
- Pathling Documentation: https://pathling.csiro.au/docs
- Delta Lake Documentation: https://docs.delta.io/
- Great Expectations: https://greatexpectations.io/

### Key Resources

**Tutorials and Examples:**
- FHIR Analytics Tutorial Slides: https://aehrc.github.io/fhir-analytics-pipeline/docs/DD23_Tutorial_230608_PiotrSzul_HowToIntegrateFhir.final.pdf
- FHIR Analytics Pipeline (GitHub): https://github.com/aehrc/fhir-analytics-pipeline
- Pathling (GitHub): https://github.com/aehrc/pathling
- OMOP-on-FHIR Examples: https://github.com/OHDSI/OMOP-on-FHIR

**Tools:**
- USAGI (Concept Mapping Tool): https://github.com/OHDSI/Usagi
- Ontoserver: https://ontoserver.csiro.au/
- Synthea (Synthetic FHIR Data): https://github.com/synthetichealth/synthea
- WhiteRabbit (Source Data Scanning): https://github.com/OHDSI/WhiteRabbit

### Research Publications

**FHIR to OMOP Conversion:**
- "Pathling: analytics on FHIR" (Journal of Biomedical Semantics, 2022): https://jbiomedsem.biomedcentral.com/articles/10.1186/s13326-022-00277-1
- "OMOP-on-FHIR: Integrating Clinical Data Through FHIR Bundle to OMOP CDM" (PubMed, 2025): https://pubmed.ncbi.nlm.nih.gov/40380541/
- "MENDS-on-FHIR: Leveraging the OMOP Common Data Model and FHIR Standards" (JAMIA Open, 2024): https://academic.oup.com/jamiaopen/article/7/2/ooae045/7685048
- "FHIR-Ontop-OMOP: Building Clinical Knowledge Graphs" (ScienceDirect, 2022): https://www.sciencedirect.com/science/article/pii/S1532046422002064

**Data Harmonization:**
- "A Consensus-based Approach for Harmonizing the OHDSI Common Data Model with HL7 FHIR" (PMC): https://pmc.ncbi.nlm.nih.gov/articles/PMC5939955/
- "Ontoserver: A Syndicated Terminology Server" (Journal of Biomedical Semantics, 2018): https://jbiomedsem.biomedcentral.com/articles/10.1186/s13326-018-0191-z

### Community Resources

**Discussion Forums:**
- OHDSI Forums: https://forums.ohdsi.org/
- HL7 FHIR Community: https://chat.fhir.org/
- FHIR Zulip Chat: https://chat.fhir.org/

**Working Groups:**
- HL7 FHIR Infrastructure WG: http://www.hl7.org/Special/committees/fiwg/
- HL7 Clinical Quality Information WG: http://www.hl7.org/Special/committees/cqi/
- OHDSI CDM WG: https://ohdsi.org/web/wiki/doku.php?id=projects:workgroups:cdm-wg

### Additional Resources

**Sample Data:**
- Synthea: https://github.com/synthetichealth/synthea
- SMART on FHIR Test Data: https://docs.smarthealthit.org/
- HL7 FHIR Examples: http://hl7.org/fhir/examples.html
- MIMIC-III Demo (for testing): https://physionet.org/content/mimiciii-demo/

**Video Resources:**
- OHDSI Tutorials: https://www.ohdsi.org/tutorial-index/
- FHIR DevDays Videos: https://www.fhir.org/devdays
- Pathling Webinars: Check CSIRO Australian e-Health Research Centre

---

## Glossary

**CDM**: Common Data Model - OMOP's standardized database schema for observational health data

**Concept**: A standardized clinical term in OMOP vocabulary with a unique concept_id

**Delta Lake**: An open-source storage layer that brings ACID transactions to Apache Spark

**ETL**: Extract, Transform, Load - The process of moving and transforming data between systems

**FHIR**: Fast Healthcare Interoperability Resources - HL7's standard for healthcare data exchange

**FHIRPath**: A query language for navigating and extracting data from FHIR resources

**NDJSON**: Newline Delimited JSON - A format where each line is a valid JSON object

**OMOP**: Observational Medical Outcomes Partnership - A common data model for observational health research

**Ontoserver**: A FHIR terminology server supporting SNOMED CT, LOINC, and other code systems

**Pathling**: An open-source tool for FHIR analytics built on Apache Spark

**PySpark**: The Python API for Apache Spark

**Standard Concept**: An OMOP vocabulary concept designated as the standard representation for a clinical idea

**Terminology Server**: A FHIR server that provides terminology services (ValueSet expansion, code translation, etc.)

**USAGI**: A tool for creating mappings between source codes and OMOP standard concepts

**Vocabulary**: A collection of clinical codes and their relationships in OMOP (e.g., SNOMED CT, LOINC, RxNorm)

---

## License and Acknowledgments

This document is based on open-source tools and standards from:
- CSIRO Australian e-Health Research Centre (Pathling, Ontoserver)
- Observational Health Data Sciences and Informatics (OHDSI/OMOP)
- HL7 International (FHIR)
- Apache Software Foundation (Spark)

For questions or contributions, please refer to the respective community forums and GitHub repositories listed in the References section.

---

**Document Version**: 2.0
**Last Updated**: 2025-01-12
**Maintained By**: CareMate Data Engineering Team
