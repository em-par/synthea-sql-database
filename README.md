
# INTRODUCTION
This project used the open source Synthea patient data simulator to generate realistic EHR data for 300 “patients”. 
The data was inspected using macOS Terminal, imported into PostgreSQL, and analyzed using SQL queries within DBeaver.


# GENERATING SYNTHETIC DATA
I downloaded Synthea locally and ran it using macOS Terminal to generate realistic EHR data

java -jar synthea-with-dependencies.jar

This generated healthcare datasets (patients, encounters, payers, etc.) 


# DATABASE SETUP
I inspected the CSV files to identify the column headings with a command in macOS Terminal.

`head -n 1 ~/Library/Mobile\ Documents/com~apple~CloudDocs/Documents/synthea/output/csv/patients.csv | tr ',' '\n' | nl`

Using DBeaver, I created tables and imported the data into PostgreSQL.
```
CREATE TABLE patients (
    id UUID PRIMARY KEY,
    birthdate DATE,
    deathdate DATE,
    ssn VARCHAR(11),
    drivers VARCHAR(50),
    passport VARCHAR(50),
    prefix VARCHAR(10),
    first VARCHAR(50),
    middle VARCHAR(50),
    last VARCHAR(100),
    suffix VARCHAR(10),
    maiden VARCHAR(100),
    marital VARCHAR(20),
    race VARCHAR(50),
    ethnicity VARCHAR(50),
    gender VARCHAR(20),
    birthplace VARCHAR(100),
    address VARCHAR(100),
    city VARCHAR(50),
    state VARCHAR(20),
    county VARCHAR(50),
    fips VARCHAR(5),
    zip VARCHAR(10),
    lat NUMERIC(10, 6),
    lon NUMERIC(10, 6),
    healthcare_expenses NUMERIC(14,2),
    healthcare_coverage NUMERIC(14,2),
    income INTEGER
);
```
```
COPY patients
FROM '/Users/emily/Documents/Synthea/output/csv/patients.csv'
DELIMITER ','
CSV HEADER;
```
I used similar CREATE TABLE and COPY statements for the other datasets (encounters, payers, providers, etc.) to build relationships between tables.


# QUERIES
I analyzed the data across clinical, organizational, and financial domains using DBeaver. 

### EXAMPLE 1
```
SELECT 
	p.race,
	y.name AS payer_name,
	count(e.id) AS encounter_count
FROM patients p 
JOIN encounters e ON e.patient = p.id
JOIN payers y ON y.id = e.payer
GROUP BY p.race, y.name, e.payer
ORDER BY y.name ASC, encounter_count DESC;
```
This query joins the patients, payers, and encounters tables to aggregate encounter data. Results are grouped by racial background and insurance payer, providing insight into payer distribution trends across patient demographics.

Below is a sample output showing encounter volume by race and payer from synthetic EHR data.

| race  | payer_name              | encounter_count |
|-------|-------------------------|-----------------|
| white | Aetna                   | 512             |
| black | Aetna                   | 177             |
| asian | Aetna                   | 19              |
| white | Anthem                  | 581             |
| black | Anthem                  | 165             |
| white | Blue Cross Blue Shield  | 962             |
| asian | Blue Cross Blue Shield  | 46              |
| black | Blue Cross Blue Shield  | 29              |


### EXAMPLE 2
```
SELECT 
    y.name AS payer_name,
    ROUND(AVG(e.total_claim_cost), 2) AS avg_claim_cost,
    ROUND(SUM(e.total_claim_cost), 2) AS total_claim_cost,
    COUNT(e.id) AS encounter_count
FROM encounters e
JOIN payers y ON y.id = e.payer
GROUP BY y.name
ORDER BY total_claim_cost DESC;
```
This query joins the encounters and payers tables to calculate the average and total claim costs per payer along with encounter counts. Results are grouped by payer and ordered by the total claim costs in descending order. 


### EXAMPLE 3
```
SELECT 
    m.description AS medication_name,
    COUNT(m.code) AS medication_count,
    ROUND(AVG(m.base_cost), 2) AS avg_cost
FROM medications m
JOIN encounters e ON e.id = m.encounter
GROUP BY m.description
ORDER BY medication_count DESC
LIMIT 10;
```
This query joins the medications and encounters tables to identify the most frequently prescribed medications and their average base costs. It aggregates medication counts and averages base costs to highlight common prescriptions across encounters. Results are grouped by medication and ordered by medication count in descending order.
