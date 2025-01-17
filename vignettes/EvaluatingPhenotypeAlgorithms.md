---
title: "EvaluatingPhenotypeAlgorithms"
author: "Joel N. Swerdel"
date: "2019-07-16"
output:
  pdf_document:
    number_sections: yes
    toc: yes
  html_document:
    number_sections: yes
    toc: yes
vignette: >
  %\VignetteIndexEntry{Evaluating Phenotype Algorithms Using Phevaluator}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding{UTF-8}

---



\newpage
# Introduction

The `Phevaluator` package enables evaluating the performance characteristics of phenotype algorithms (PAs) using data from databases that are translated into the Observational Medical Outcomes Partnership Common Data Model (OMOP CDM).

This vignette describes how to run the PheValuator process from start to end in the `Phevaluator` package. 

# Overview of Process 

There are several steps in performing a PA evaluation:
1. Creating the extremely specific (xSpec), extremely sensitive (xSens), and prevalence cohorts
2. Creating the Diagnostic Predictive Model using the PatientLevelPrediction (PLP) package
3. Creating the Evaluation Cohort
4. Creating the Phenotype Algorithms for evaluation
5. Evaluating the PAs
6. Examining the results of the evaluation

Each of these steps is described in detail below.  For this vignette we will describe the evaluation of PAs for diabetes mellitus (DM).

## Creating the Extremely Specific (xSpec), Extremely Sensitive (xSens), and Prevalence Cohorts

The extremely specific (xSpec), extremely sensitive (xSens), and prevalence cohorts are developed using the ATLAS tool.  The xSpec is a cohort where the subjects in the cohort are likely to be positive for the health outcome of interest (HOI) with a very high probability.  This may be achieved by requiring that subjects have multiple condition codes for the HOI in their patient record.  An [example](http://www.ohdsi.org/web/atlas/#/cohortdefinition/1769699) of this for DM is included in the OHDSI ATLAS repository.  In this example each subject has an initial condition code for DM.  The cohort definition further specifies that each subject also has a second code for DM between 1 and 30 days after the initial DM code and 10 additional DM codes in the rest of the patient record.  This very specific algorithm for DM ensures that the subjects in this cohort have a very high probability for having the condition of DM.  This PA also specifies that subjects are required to have at least 365 days of observation in their patient record.

An example of an xSens cohort is created by developing a PA that is very sensitive for the HOI.  The system uses the xSens cohort to create a set of "noisy" negative subjects, i.e., subjects with a high likelihood of not having the HOI.  This group of subjects will be used in the model building process and is described in detail below.  An [example](http://www.ohdsi.org/web/atlas/#/cohortdefinition/1770120) of an xSens cohort for DM is also in the OHDSI ATLAS repository.

An example of an prevalence cohort is created by developing a PA that is very sensitive for the HOI.  The system uses the prevalence cohort to provide a reasonable approximation of the prevalence of the HOI in the population. This improves the calibration of the predictive model.  This group of subjects will be used in the model building process and is described in detail below.  An [example](http://www.ohdsi.org/web/atlas/#/cohortdefinition/1770119) of an prevalence cohort for DM is also in the OHDSI ATLAS repository.

## Creating the Diagnostic Predictive Model

The function CreatePhenoModel develops the diagnostic predictive model for assessing the probability of having the HOI in the evaluation cohort. 

CreatePhenoModel should have as inputs:

 * connectionDetails \- connectionDetails created using the function createConnectionDetails in the DatabaseConnector package.
 * xSpecCohort \- The number of the "extremely specific (xSpec)" cohort definition id in the cohort table (for noisy positives)
 * cdmDatabaseSchema \- The name of the database schema that contains the OMOP CDM instance. Requires read permissions to this database. On SQL Server, this should specify both the database and the schema, so for example 'cdm_instance.dbo'.
 * cohortDatabaseSchema \- The name of the database schema that is the location where the cohort data used to define the at risk cohort is available. If cohortTable = DRUG_ERA, cohortDatabaseSchema is not used by assumed to be cdmSchema. Requires read permissions to this database.
 * cohortDatabaseTable \- The tablename that contains the at risk cohort. If cohortTable <> DRUG_ERA, then expectation is cohortTable has format of COHORT table: cohort_concept_id, SUBJECT_ID, COHORT_START_DATE, COHORT_END_DATE.
 * outDatabaseSchema \- The name of a database schema where the user has write capability.  A temporary cohort table will be created here.
 * trainOutFile \- A string designation for the training model file
 * exclCohort \- The number of the "extremely sensitive (xSens)" cohort definition id in the cohort table (used to estimate population prevalence and to exclude subjects from the noisy positives)
 * prevCohort \- The number of the cohort definition id to determine the disease prevalence, usually a super-set of the exclCohort
 * estPPV \- A value between 0 and 1 as an estimate for the positive predictive value for the exclCohort
 * modelAnalysisId \- Another string for designating the name for the model files
 * excludedConcepts \- A list of conceptIds to exclude from featureExtraction which should include all concept_ids used to create the xSpec and xSens cohorts
 * cdmShortName \- A short name for the current database (CDM)
 * mainPopnCohort \- The number of the cohort to be used as a base population for the model (default=NULL)
 * lowerAgeLimit \- The lower age for subjects in the model (default=NULL)
 * upperAgeLimit \- The upper age for subjects in the model  (default=NULL)
 * startDate \- The starting date for including subjects in the model (default=NULL)
 * endDate \- The ending date for including subjects in the model (default=NULL)

The CreatePhenoModel function creates a PLP model to be used for determining the probability of the HOI in the evaluation cohort.  

For example:


```r
setwd("c:/phenotyping")

connectionDetails <- createConnectionDetails(dbms = "postgresql",
                                              server = "localhost/ohdsi",
                                              user = "joe",
                                              password = "supersecret")

phenoTest <- PheValuator::createPhenoModel(connectionDetails = connectionDetails,
                           xSpecCohort = 1769699,
                           cdmDatabaseSchema = "my_cdm_data",
                           cohortDatabaseSchema = "my_results",
                           cohortDatabaseTable = "cohort",
                           outDatabaseSchema = "scratch.dbo", #a database schema with write access
                           trainOutFile = "PheVal_10X_DM_train",
                           exclCohort = 1770120, #the xSens cohort
                           prevCohort = 1770119, #the cohort for prevalence determination
                           estPPV = 0.75,
                           modelAnalysisId = "20181206V1", 
                           excludedConcepts = c(201820), 
                           cdmShortName = "myCDM", 
                           mainPopnCohort = 0, #use the entire subject population
                           lowerAgeLimit = 18, 
                           upperAgeLimit = 90,
                           startDate = "20100101",
                           endDate = "20171231")
```
In this example, we used the cohorts developed in the "my_results" cdm, specifying the location of the cohort table (cohortDatabaseSchema, cohortDatabaseTable - "my_results.cohort") and where the model will find the conditions, drug exposures, etc. to inform the model (cdmDatabaseSchema - "my_cdm_data").  The subjects included in the model will be those whose first visit in the CDM is between January 1, 2010 and December 31, 2017.  We are also specifically excluding the concept ID 201826, "Type 2 diabetes mellitus" which was used to create the xSpec cohort.  Their ages at the time of first visit will be between 18 and 90. With the parameters above, the name of the predictive model output from this step will be: "c:/phenotyping/lr_results_PheVal_10X_DM_train_myCDM_ePPV0.75_20181206V1.rds"

## Creating the Evaluation Cohort

The function CreateEvalCohort uses the PLP function applyModel to produce a large cohort of subjects, each with a predicted probability for the HOI.

For example:


```r
setwd("c:/phenotyping")

connectionDetails <- createConnectionDetails(dbms = "postgresql",
                                              server = "localhost/ohdsi",
                                              user = "joe",
                                              password = "supersecret")

evalCohort <- PheValuator::createEvalCohort(connectionDetails = connectionDetails,
                              xSpecCohort = 1769699, 
                              cdmDatabaseSchema = "my_cdm_data",
                              cohortDatabaseSchema = "my_results",
                              cohortDatabaseTable = "cohort",
                              outDatabaseSchema = "scratch.dbo",
                              testOutFile = "PheVal_10X_DM_eval",
                              trainOutFile = "PheVal_10X_DM_train",
                              estPPV = 0.75,
                              modelAnalysisId = "20181206V1", 
                              evalAnalysisId = "20181206V1",
                              cdmShortName = "myCDM", 
                              mainPopnCohort = 0, 
                              lowerAgeLimit = 18, 
                              upperAgeLimit = 90,
                              startDate = "20100101",
                              endDate = "20171231")
```
In this example, the parameters specify that the function should use the model file:
  "c:/phenotyping/lr_results_PheVal_10X_DM_train_myCDM_ePPV0.75_20181206V1.rds"
to produce the evaluation cohort file:
  "c:/phenotyping/lr_results_PheVal_10X_DM_eval_myCDM_ePPV0.75_20181206V1.rds"
The evaluation cohort file above will be used the evaluation of the PAs provided in the next step.

## Creating the Phenotype Algorithms for evaluation

The next step is to create the PAs o be evaluated.  These are specific to the research question of interest.  For certain questions, a very sensitive algorithm may be required; others may require a very specific algorithm.  For this example, we will test an algorithm which requires that the subject have a diagnosis code for DM from an in-patient setting where the code was specified as the primary reason for discharge.  An [example](http://www.ohdsi.org/web/atlas/#/cohortdefinition/1769702) of this algorithm is in the OHDSI ATLAS repository. The output of this function is a list containing 2 data frames, one with the results of the PA evaluation and a second with a set of subject IDs that were determined to be true positives, false positives, or false negatives based on prediction threshold of 50%.  A true positive, with this criteria, would be a subject that was included in the PA and also had a predicted value for the HOI of 0.5 or greater.  A false positive would be a subject who was included in the PA and whose predicted probability was less than 0.5.  A false negative would be a subject who was not included in the PA but had a predicted probability of the HOI or 0.5 or greater.

For example:


```r
setwd("c:/phenotyping")

connectionDetails <- createConnectionDetails(dbms = "postgresql",
                                              server = "localhost/ohdsi",
                                              user = "joe",
                                              password = "supersecret")

phenoResult <- PheValuator::testPhenotype(connectionDetails = connectionDetails,
               cutPoints = c(0.1, 0.2, 0.3, 0.4, 0.5, "EV", 0.6, 0.7, 0.8, 0.9),
               resultsFileName = "c:/phenotyping/lr_results_PheVal_10X_DM_eval_myCDM_ePPV0.75_20181206V1.rds",
               cohortPheno = 1769702,
               phenText = "All Diabetes by Phenotype 1 X In-patient, 1st Position",
               order = 1,
               testText = "Diabetes Mellitus xSpec Model - 10 X T2DM",
               cohortDatabaseSchema = "my_results",
               cohortTable = "cohort",
               estPPV = 0.75, 
               cdmShortName = "myCDM")
```
In this example, a wide range of prediction thresholds are provided (cutPoints) including the expected value ("EV").  Given that parameter setting, the output from this step will provide performance characteristics (i.e, sensitivity, specificity, etc.) at each prediction threshold as well as those using the expected value calculations as described in the [Step 2 diagram](vignettes/Figure2.png).  The evaluation uses the prediction information for the evaluation cohort developed in the prior step.  The data frames produced from this step may be saved to a csv file for detailed examination.







