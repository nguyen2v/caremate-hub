
## Analytic
### Objective
* build a simple, automated, batch analytics pipeline for FHIR data using opensource software, including Apache Spark and Pathling. 
* use FHIRPath and SQL to transform complex FHIR data into simpler, use-case centric tabular views 
* to tackle some of the complexities of working with FHIR data, including: codes and terminology, Questionnaires, and extensions


### Scenario

![](https://i.imgur.com/1HbS1nM.png)

![](https://i.imgur.com/LshlPU0.png)


#### FHIR mapping to SQL
![](https://i.imgur.com/R2wsjJF.png)







### Implementation

Get sample data from [[HL7 Sample Data]]
#### Technologies 
	• Apache Spark, Spark SQL, PySpark, 
	• Pathling and Ontoserver (for terminology queries) 
	• ndjson, parquet/delta 
	• Python, SQL, FHIRPath 
####  Databricks (for convenience) 
	• Spark cluster for execution of the pipeline code 
	• Notebooks for exploratory examples 
	• Data flow orchestration + SQL Analytics and reporting
## Appendix

#### REFERENCE
* Slides in Devday: https://aehrc.github.io/fhir-analytics-pipeline/docs/DD23_Tutorial_230608_PiotrSzul_HowToIntegrateFhir.final.pdf
• Github: 
	• The Pipeline: https://github.com/aehrc/fhir-analytics-pipeline 
	• Pathling: https://github.com/aehrc/pathling

