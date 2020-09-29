# Building a Demographically Accurate Synthetic Patient Electronic Medical Record Generation Pipeline

## Introduction

Medical data typically contains very sensitive information about the patients, this coupled with an increase in privacy rules surrounding access
and utilization of data makes it especially difficult to access real patient electronic health records. 

Another difficulty associated with access to real medical data is the need to properly anonymize the data by eliminating any links to the actual patient. There have been cases <a href='#ref1'>[1]</a>  where  actual patient recovery from anonymized data was indeed possible.

These factors have led a number of research efforts to resort to the use of synthetically generated patient records using data from publicly available medical databases such as [PubMed](https://pubmed.ncbi.nlm.nih.gov/) or other sources such as medical literature,  published records of disease symptom relationships etc.

## The Objective

The intended objective was to identify/build/design a solution which could be used to generate as realistic-as-possible patient electronic medical records to support a research project on the use of Machine Learning techniques in the generation of differential diagnosis based on patient medical records.

The selected method for generating this data should have the following characteristics:

- Transparent and Reproducible Generation process and
- Incorporate Demographic Information.


## The Solution

### Synthea: A Synthetic Patient Generator

The first component in the search for synthetic data was [Synthea](https://github.com/synthetichealth/synthea). Written in Java, Synthea - a Synthetic Patient Population Simulator - allows for the generation of *realistic* patient medical records. To avoid privacy concerns, the generator was developed relying on publicly available medical information and health statistics. The source code for the project is also fully open-source with a permissive license which allows prospective users to modify the codebase to suit target applications. In its earliest version, Synthea modeled conditions ranked as the top 10 causes of visits to a primary health care provider and the top 10 conditions according to the "*years of life lost*" metric. Since this initial version, support has been added for more conditions with many of these contributed by its active community. This contribution is largely due to an expressive generic module framework which allows even non-technical users (with sufficient medical knowledge) to model any medical condition.

A high-level graphical overview of Synthea's architecture can be found [here](https://github.com/synthetichealth/synthea/wiki/Architecture).

Key to the selection of Synthea is its [Generic Module Framework](https://github.com/synthetichealth/synthea/wiki/Generic-Module-Framework). This framework provides a rich toolkit for the modeling of diseases, conditions, procedures etc. which might affect a patient and contribute to medical records for said patient. It allows incorporation of statistics regarding the diseases, symptoms, etc in a generic modules which in turn serve as input for the Synthea generator and allow it to generate realistic data samples i.e. synthetic patients with believable medical records.

#### **The Synthea Limitation**

While Synthea does generate the patient records, more focus is placed on activities carried out during encounters with the health care providers e.g. laboratory procedures, payments, prescribed medication, etc. Very little attention is placed on the symptomatic expression of a particular condition. For many of the disease modules included by default in Synthea (at the time of writing), symptoms were left out of the disease models and in the few cases where they were included, there was no statistical relationship between the condition being modeled and the symptoms presented. 

However, Synthea's expressive generic module framework guarantees that provided a source of the probabilistic relationship between Symptoms and conditions is available then modules could be created which are able to capture this relationship.


### Symcat

[Symcat](http://www.symcat.com/) is described by its creators as:

> A disease calculator that uses hundreds of thousands of patient records to estimate the probability of disease.

It provides an interface where users can supply information about the symptoms being experienced and receive a differential diagnosis. Of greater importance to this project, however, is Symcat's [conditions](http://www.symcat.com/conditions) and [symptoms](http://www.symcat.com/symptoms) directory. This knowledgebase which is publicly available on Symcat's website provides probabilistic relationships between symptoms and conditions. It also includes the prevalence of diseases and  symptoms by age, gender, race, and ethnicity.

A scrapped version is also available in the [Symcat-Synthea](https://github.com/teliov/symcat-to-synthea/tree/master/symcat) github repository.

#### **Symcat Data Description**

Symcat contains 801 conditions and 474 symptoms. For each condition, it gives a list of symptoms presented by patients with the condition and indicates the probability that patients will present with the specified symptom. Symcat data for each condition also contains the gender-based odds of contracting the disease. Also included are race-based odds for disease contraction. Symcat contains 4 race divisions:

* White
* Black
* Hispanic and 
* Others 

Finally, age based odds for contracting each condition are provided. Symcat has 8 age groups: 

* < 1 year
* 1-4 years
* 5-14 years
* 15-29 years
* 30-44 years
* 45-59 years
* 60-74 years and
* \> 75 years

It should be stated that of the 474 symptoms, only 376 of them were associated with a condition. Symptoms with no condition association were dropped.

## Data Generation Pipeline

### Parsing Symcat Data

The Symcat conditions and symptoms data (mentioned above) were first processed to extract relevant information in a more structured format. The [Symcat-to-Synthea repository](https://github.com/teliov/symcat-to-synthea) contains a parser which serves this purpose.

To parse the data, the following steps should be followed:

```bash
# Parse symptoms

cd <path-to-repos>/symcat-to-synthea
./main.py --parse_symptoms --symptoms_csv symcat/symcat-474-symptoms.csv --output output/

# Parse conditions
./main.py --parse_conditions --conditions_csv symcat/symcat-801-diseases.csv --output output/
```

The commands in the snippet above produced json exports - symptoms.json and conditions.json respectively - which contain the parsed data in a more structured format. These files then serve as input into the module generation phase of the pipeline.

### Generating Synthea Modules

As mentioned previously, Synthea allows the modeling of conditions using generic modules. Also recall that the main aim of including Symcat was to replace the missing condition-symptom probabilistic relationships present by default in Synthea.

To this end, a generator was created which is able to take as input the `conditions.json` and `symptoms.json` output files mentioned above - which contain the condition and symptom definitions, the probabilistic relationship between them and their dependence on patient demography (age, sex, race) - and generate valid Synthea Modules.

To generate these modules - assuming the steps for parsing Symcat data above have been followed - the following command is run:

``` bash
# generate synthea modules

cd <path-to-repos>/symcat-to-synthea
./main.py --gen_modules --symptoms_json output/symptoms.json --conditions_json output/conditions.json --output output/modules --generator_mode 1 --prefix symcat_
```

Once this command is complete the `output/modules` directory would contain the generated modules.

#### **Module Generator Configuration Options**

There are a number of configuration options which can be passed to the generator script. The [extended-tareq-parser](https://github.com/teliov/symcat-to-synthea/tree/extended-tareq-parser) branch of the symcat-to-synthea repo has a more detailed description of the options available. Nonetheless a brief summary is included here:

**--generator-mode**: This determines how the probabilities from Symcat's data are utilized when generating the disease modules. The basic mode (used in the sample above) uses a Naive Bayes approach to determine the probability of a patient contracting a disease based on age, sex and race. The symptoms expressed by the patient are then selected based on the probabilistic relationship between the condition and the symptoms. Symptoms with higher expression probabilities are more likely to be expressed and vice versa. The advanced generation mode (the default setting), attempts to also use demographic statistics for symptoms when selecting which symptoms would be expressed. In this mode, in addition to the probabilistic relationship between the condition and the symptom, the relationship between the symptoms expression and the patient's age, sex and race is also utilized. Also in determining the condition contraction probability, prior information (extracted from census data) is utilized. For the project, the basic mode was used. While the advanced mode seeks to model the more complex relationship that exists between conditions, symptoms and patient demography there were [issues](https://github.com/teliov/symcat-to-synthea/issues/15) (e.g for instance negative probability values) which required some assumptions/approximations to correct, hence the fallback to the basic mode.

**--num_history_years**:
While generating data with Synthea, it was observed that the age distribution of the patients with a given condition was skewed to a very young age. This was due to a number of factors:

- Synthea runs a patient through an entire life cycle (from birth to death) and the disease modules created were non terminal - meaning that as long as a patient was alive it was possible to recontract a disease. This greatly slowed down the generation process.

- In a bid to mitigate this, a limit was placed on how many times a patient could contract a disease, but since during the lifecycle simulation process, the generator attempts multiple times to assign a disease to a patient, this limit was reached quite early in the patient's life.

As a result, the `num_history_years` - n -  prevents the generator from attempting to simulate the patient's probablity of contracting diseases until the patient is `maxAge - n` years old. This value defaults to 1, and results in a age distribution that is more realistic.

**--min_symptoms**:
Since the symptoms expressed by a sick patient are selected probabilistically, it is possible that a sick patient presents with no symptom. This configuration ensures that a minimum number of symptoms is presented by a sick patient. The default is set to 1. Once the generation process is complete and the number of symptoms included are not up to the minimum number of specified symptoms, then the generator automatically inserts symptoms with the most likely symptom being inserted first until the minimum number is reached.

**--prefix**:
This option allows the specification of a prefix for the generated modules. For this project the prefix *symcat_* was used. In this case, given a disease e.g. *appendicitis* then the generated module would be *symcat_appendicitis*. This allows an easy distinction between modules generated using Symcat data and Synthea's default modules. The importance of this distinction would be apparent in the section on generating data.

#### **Synthea Version**

To reproduce the data generated during this project, [this forked version of Synthea](https://github.com/teliov/synthea) should be used. Slight modifications were made to the format of the exported data. This version is also behind the current master version of Synthea (as at the time of writing), hence using this forked version would reduce the chances of errors or mismatch due to changes introduced in the repo.

#### **Running Synthea**

There are a number of ways to generate the data using Synthea.

1. Synthea includes a `run_synthea` script. This requires that the script be run from the Synthea repository folder.

2. A `.jar` file can be generated with all dependencies bundled and this can be run on any platform which supports Java. In the event that the target platform does not support Java and it is not possible to install Java (as was the case during this project), a [singularity container](https://github.com/teliov/synthea-singularity) was developed which creates a Java container capable of executing the Synthea jar file. The command below can be used to generate the required jar file (located in the `<synthea-repo>/build/libs` directory):
```bash
cd <synthea-repo>
./gradlew uberJar
```

The command for generating data is given below:
```bash
cd <synthea-repo>
./run_synthea -p <num_patients> --exporter.fhir.export=false\
    --exporter.practitioner.fhir.export=false\
    --exporter.symptoms.csv.export=true\
    --exporter.symptoms.mode=1\
    --exporter.years_of_history=0\
    -m symcat_*\
    -d <path-to-generated-modules>\
    --exporter.baseDirectory=<path-to-data-output>
```

When running using a generated jar file, the `./run_synthea` part of the command above is replaced with:
```bash
java -jar <path-to-generated-jar>
```

All the options remain the same.

**Run-Through of Active Options**

**-p <num_patients>**: This specifies the number of patients to be generated. Note that since one patient can have multiple conditions, it does not determine the number of diagnosed conditions which would be generated. However, the number of conditions generated is positively correlated with the number of patients specified.

**--exporter.practitioner.fhir.export=false**: This disables - in this case - an *[FHIR](http://hl7.org/fhir/)* export of generated patients. For this project, the desired output was CSV and Synthea defaults to *FHIR*.

**--exporter.symptoms.mode=1**: This instructs Synthea to create a CSV file of the symptoms. By default this is turned off i.e. set to 0.

**--exporter.years_of_history=0**: This instructs Synthea to export data for a patients entire lifetime. By setting a non-negative value *n*, it is possible to instruct Synthea to restrict the export to the last *n* available years of the patient's data.

**-m symcat_**: This specifies a module regex which Synthea uses in selecting which disease modules would be active during the generation process. By specifying *symcat_* we can target only those modules generated using the `symcat-to-synthea` generator as described earlier. Synthea's default modules are thus excluded.

**-d \<path-to-generated-modules\>**: This specifies the path to the generated modules. By default Synthea  bundles its own modules (e.g in the generated jar file) and uses those when generating. This flag instructs the generator to search for modules in the specified directory.

**--exporter.baseDirectory=\<path-to-data-output\>**: This specifies the output location of generated data.

A sample output directory structure is shown below:
```
./output
└── symptoms
    └── csv
        └── symptoms.csv

2 directories, 1 file 
```

A sample from the generated file is shown below:
```
PATIENT,GENDER,RACE,ETHNICITY,AGE_BEGIN,AGE_END,PATHOLOGY,NUM_SYMPTOMS,SYMPTOMS
03a8161d-6647-4556-94eb-4622b8f7fcb7,M,black,nonhispanic,3,3,48c043a01ab7bed4da9957a8900ba1971c78031d06abdf80739f9012,3,8ecefd608d13429302287e01f56ca67ffe8800643c021a8080ea291d;36c572503a8f8b783f0da9493c35a8770c9076d6506c63b6eef37e71;23aa4195d51eb504527500d89651b974ff771aa140a2b204c631682f
```

- *03a8161d-6647-4556-94eb-4622b8f7fcb7* corresponds to the patient ID
- *48c043a01ab7bed4da9957a8900ba1971c78031d06abdf80739f9012* corresponds to the *sha224* hash of the condition.
- The `;` separated hexadecimal UUIDs correspond to the sha224 hashes of the symptoms expressed by this patient.

The map between a hash and it's actual value can be extracted from the generated `conditions.json` and `symptoms.json` files.

## References

* <span id="ref1">[1]</span> Latanya Sweeney, Akua Abu, Julia Winn (2013). Identifying Participants in the Personal Genome Project by Name
