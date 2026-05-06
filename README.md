# 🌍 Infrastructure Data Platform sur GCP avec Terraform

## 📋 Contexte du projet

Mise en place d'une infrastructure cloud complète pour un projet d'analyse de données financières (ESG - critères Environnementaux, Sociaux et de Gouvernance). L'objectif est de créer une plateforme de données scalable permettant l'ingestion, le stockage et l'analyse de données de fonds d'investissement.

### 🎯 Objectifs

- Provisionner une infrastructure reproductible avec **Terraform**
- Mettre en place une architecture Medallion (raw → staging → marts)
- Stocker les données brutes dans **Cloud Storage**
- Créer des tables externes dans **BigQuery**
- Préparer l'environnement pour **dbt** (transformations)

---

## 🏗️ Architecture cible
```
┌─────────────────────────────────────────────────────────────┐
│                      INFRASTRUCTURE GCP                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  📁 Cloud Storage                                            │
│  │                                                           │
│  └── raw/                                                    │
│      └── funds.csv ←─── Fichiers sources                     │
│                                                              │
│  ▼                                                           │
│  📊 BigQuery                                                 │
│  ├── raw (bronze) ← Tables externes                          │
│  ├── staging (silver) ← Données nettoyées                    │
│  └── marts (gold) ← Agrégats finaux                          │
│                                                              │
│  🔐 Service Account: dbt-sa-clean                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```
---
## 🚀 Étape 1 : Créer le projet GCP

```bash
# Via la console Google Cloud
# 1. Aller sur console.cloud.google.com
# 2. Cliquer sur "NEW PROJECT"
# 3. Nom du projet : data-eng-clean
# 4. ID : data-eng-clean
# 5. Cliquer sur "CREATE"
# Via gcloud CLI (alternative)
gcloud projects create data-eng-clean --name="Data Engineering Clean"
gcloud config set project data-eng-clean
```
## 🚀 Étape 2 : Créer la structure Terraform
# Créer le dossier du projet
```bash
  mkdir terraform_clean
  cd terraform_clean
```
## main.tf - Ressources principales
```hcl
 terraform {
  required_version = ">= 1.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Bucket Cloud Storage
resource "google_storage_bucket" "bucket" {
  name          = "data-lake-${var.project_id}"
  location      = var.region
  force_destroy = true
  uniform_bucket_level_access = true
}

# Dataset RAW (bronze)
resource "google_bigquery_dataset" "raw" {
  dataset_id = "raw"
  friendly_name = "Raw Data Layer"
  location = var.region
}

# Dataset STAGING (silver)
resource "google_bigquery_dataset" "staging" {
  dataset_id = "staging"
  friendly_name = "Staging Data Layer"
  location = var.region
}

# Dataset MARTS (gold)
resource "google_bigquery_dataset" "marts" {
  dataset_id = "marts"
  friendly_name = "Marts Data Layer"
  location = var.region
}

# Service Account pour dbt
resource "google_service_account" "dbt_sa" {
  account_id   = "dbt-sa-clean"
  display_name = "dbt Service Account"
}

# Permissions BigQuery
resource "google_project_iam_member" "dbt_bigquery_job" {
  project = var.project_id
  role    = "roles/bigquery.jobUser"
  member  = "serviceAccount:${google_service_account.dbt_sa.email}"
}

resource "google_project_iam_member" "dbt_bigquery_editor" {
  project = var.project_id
  role    = "roles/bigquery.dataEditor"
  member  = "serviceAccount:${google_service_account.dbt_sa.email}"
}

# Permission Storage
resource "google_project_iam_member" "dbt_storage_viewer" {
  project = var.project_id
  role    = "roles/storage.objectViewer"
  member  = "serviceAccount:${google_service_account.dbt_sa.email}"
}
```
## variables.tf - Variables
```hcl
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "GCP Region"
  type        = string
  default     = "europe-west6"
}
```
## terraform.tfvars - Valeurs
```hcl
project_id = "data-eng-clean"
region     = "europe-west6"
```
## 🚀 Étape 3 : Exécuter Terraform
```bash
# Authentification GCP
gcloud auth application-default login
gcloud config set project data-eng-clean

# Initialisation de Terraform
terraform init

# Vérification du plan
terraform plan

# Déploiement de l'infrastructure
terraform apply -auto-approve
```
## 🚀 Étape 4 : Upload des données CSV
### 4.1 Création du fichier CSV (exemple)
```bash
@'
fund_id,fund_name,legal_structure,domicile,esg_score,sfdr_article,aum_millions,currency,inception_date,management_fee,risk_level,asset_class,investment_style,is_ucits,is_green_taxonomy
LUX001,AB Sustainable Global Thematic,SICAV,Luxembourg,92,Article 9,1250,EUR,2020-03-15,0.75,3,Equity,Thematic,TRUE,TRUE
LUX002,BCEE Climate Action Fund,FCP,Luxembourg,88,Article 9,875,EUR,2019-06-01,0.85,4,Equity,Active,TRUE,TRUE
LUX003,BNP Paribas Easy ESG ETF,SICAV,Luxembourg,76,Article 8,2340,EUR,2018-11-20,0.35,5,Equity,Passive,TRUE,FALSE
LUX004,China Green Transition,SRAICAV,Switzerland,82,Article 8,420,CHF,2021-01-10,1.20,6,Equity,Active,FALSE,FALSE
LUX005,Dexia Sustainable Bonds,SICAV,Luxembourg,71,Article 8,1890,EUR,2017-09-05,0.55,3,Fixed Income,Active,TRUE,FALSE
LUX006,European Green Deal Fund,FCP,Luxembourg,95,Article 9,3100,EUR,2020-12-01,0.90,4,Infrastructure,Active,TRUE,TRUE
LUX007,Fidelity Sustainable Tech,SICAV,Ireland,84,Article 8,2670,USD,2019-04-18,0.95,6,Equity,Growth,TRUE,FALSE
LUX008,Generali ESG Balanced,SICAV,Luxembourg,68,Article 8,950,EUR,2016-08-22,1.10,4,Balanced,Multi-asset,TRUE,FALSE
LUX009,HSBC Green Bonds,SICAV,Luxembourg,79,Article 8,1520,EUR,2018-02-14,0.50,3,Fixed Income,Passive,TRUE,FALSE
LUX010,JPM Carbon Transition,SICAV,Luxembourg,73,Article 8,3450,USD,2020-07-07,0.80,5,Equity,Value,TRUE,FALSE
LUX011,BNP Paribas Climate Impact,FCP,Luxembourg,91,Article 9,680,EUR,2021-06-30,1.15,5,Equity,Thematic,TRUE,TRUE
LUX012,ClearBridge Sustainability,SICAV,Ireland,67,Article 8,540,USD,2015-11-11,1.25,6,Equity,Growth,TRUE,FALSE
LUX013,Eastspring ESG Asia,SRAICAV,Singapore,59,Article 6,310,SGD,2019-09-19,1.35,7,Equity,Active,FALSE,FALSE
LUX014,Franklin Templeton Climate,SICAV,Luxembourg,74,Article 8,1120,EUR,2017-05-03,0.88,5,Equity,Value,TRUE,FALSE
LUX015,GS Green Infrastructure,SICAV,Luxembourg,86,Article 9,890,EUR,2020-10-12,1.05,6,Infrastructure,Active,TRUE,TRUE
LUX020,AXA WF ESG Global,SICAV,Luxembourg,72,Article 8,2780,EUR,2016-12-16,0.70,4,Equity,Blend,TRUE,FALSE
LUX022,Candriam Sustainable Eq,SICAV,Luxembourg,83,Article 8,1950,EUR,2018-04-25,0.92,5,Equity,Active,TRUE,FALSE
LUX025,Edmond de Rothschild,SICAV,Luxembourg,77,Article 8,630,EUR,2019-11-08,1.30,4,Fixed Income,Active,TRUE,FALSE
LUX028,Flossbach von Storch,SICAV,Luxembourg,64,Article 6,3420,EUR,2014-02-27,0.65,5,Multi-asset,Defensive,TRUE,FALSE
LUX030,MainFirst ESG Leaders,SICAV,Luxembourg,81,Article 8,720,EUR,2020-05-14,1.40,6,Equity,Growth,TRUE,FALSE
'@ | Out-File -FilePath funds2025.csv -Encoding ascii
```
### 4.2 Upload vers Cloud Storage
```bash
# Récupérer le nom du bucket
BUCKET_NAME=$(terraform output -raw bucket_name)

# Upload du fichier
gsutil cp funds.csv gs://${BUCKET_NAME}/raw/
```
### 4.3 Création de la table externe BigQuery
```bash
# Création de la table externe

gcloud storage cp funds2025.csv gs://data-lake-data-eng-clean/funds/raw/funds2025.csv
bq mk --table --external_table_definition=@CSV=gs://data-lake-data-eng-clean/raw/funds2025.csv raw.funds2025
```
### 4.4 Vérification
```bash
# Lister les tables
bq ls raw

# Interroger les données
bq query --nouse_legacy_sql "SELECT * FROM raw.funds LIMIT 5"
```
## 📊 Structure finale
```text
data-eng-clean (projet GCP)
├── Cloud Storage
│   └── data-lake-data-eng-clean/
│       └── raw/
│           └── funds.csv
├── BigQuery
│   ├── raw/
│   │   └── funds (table externe)
│   ├── staging/ (prêt pour dbt)
│   └── marts/ (prêt pour dbt)
└── IAM
    └── dbt-sa-clean@data-eng-clean.iam.gserviceaccount.com
```
## Vider le cache DNS
```powershell
# Vider le cache DNS Windows
ipconfig /flushdns

# Redémarrer le service DNS client
Restart-Service -Name "Dnscache" -Force
```
## Utiliser les DNS Google
```powershell
# Modifier les serveurs DNS (administrateur requis)
netsh interface ip set dns "Wi-Fi" static 8.8.8.8
netsh interface ip add dns "Wi-Fi" 8.8.4.4 index=2
```
### Redémarrer PowerShell et réauthentifier
```powershell
# Quitter et rouvrir PowerShell (en tant qu'administrateur)

# Reconnecter gcloud
gcloud auth application-default login

# Refixer le projet
gcloud config set project data-eng-clean

# Relancer dbt
cd C:\Terraform_dbt_gcp\dbt_bgquery_v2\dbt_bgquery_v2
dbt run
```
### Vérifier la connexion de base
```powershell
# Tester la connexion à Google
ping 8.8.8.8

# Tester la résolution DNS
nslookup bigquery.googleapis.com

# Tester via gcloud
bq ls --project_id=data-eng-clean
```
