---
title: 'Reusable RDM Planning Environments for Trainings and Workshops: A BioHackathon Europe 2024 Report'
tags:
  - 
authors:
  - name: Jana Martínková
    orcid: 0000-0001-8575-6533
    affiliation: 1
  - name: Kryštof Komanec
    orcid: 0000-0003-3856-1682
    affiliation: 1
affiliations:
  - name: Czech Technical University in Prague
    index: 1
date: 8 November 2024
bibliography: paper.bib
group: ...
authors_short: J. Martínková, K.Komanec
git_url: https://github.com/ds-wizard/bh2024-report
event: BH2024
---

# Abstract

%% TODO Kryštof -  Write in the end

# Introduction

%% TODO Kryštof

# Content

# DSW
%% TODO Kryštof - longer description of the tool, add maybe some diagram showing/explaining how it works together (the flow of creating DMP)

Data Stewardship Wizard (DSW) is a comprehensive platform for RDM planning, offering a range of components that facilitate structured data management:
- **Knowledge Models**: These are templates of RDM best practices and guidelines, central to each DSW instance.
- **Document Templates**: These templates generate structured outputs, such as PDF plans or reports, derived from RDM plans.
- **Questionnaires**: A questionnaire is a data structure that collects responses based on the knowledge model.
- **Users and Locales**: The DSW instance also stores user profiles and localisation data for supporting multiple languages.

Each of these resources is stored in a relational database and may include assets stored in object storage (e.g., MinIO for S3-compatible storage). However, manually setting up these resources and resolving their interdependencies is challenging. For instance, knowledge models depend on user definitions, document templates, and questionnaire responses to operate cohesively. This complexity creates significant hurdles in replicating DSW setups, especially in training environments.

# Data Seeder

%% TODO Kryštof - exaplin what it is, for what

# Problem Statement

The DSW team frequently conducts training and workshop sessions to demonstrate the platform’s various features and how DSW integrates with different roles throughout the research data management lifecycle. For each event, a new DSW instance is created for participants to use during demonstrations and hands-on activities. However, each newly created instance initially lacks any content, making it unsuitable for training or workshop use straight away. To make the instance functional, relevant content must first be imported. This process involves gathering selected content resources from another instance(s) and using the DSW’s Data Seeder to populate the instance.

![Using Data Seeder](/figures/seed_maker_diagram.jpg)

Preparing data for the Data Seeder, referred to here as a content package, is currently a manual, time-intensive, and resource-heavy process. Different content resources (such as knowledge models, document templates, etc.) are interdependent and may rely on other, often non-visible, resources. Content package creators must ensure that all dependent resources are included and correctly ordered. Content insertion order is critical due to relational dependencies—adding an item with a foreign key dependency on another item that has not yet been inserted will result in an error. Additionally:

* **Identifiers** associated with instances must be altered to avoid conflicts with the original instance.
* **Author** information for items may need to be anonymised for privacy and security reasons.

Given the rising interest in DSW and the growing demand for training, this manual process is becoming increasingly unsustainable.

To address these challenges, we developed a tool, seed-maker, that automates the creation of content packages, ready for use with the DSW Data Seeder in just minutes. This tool resolves dependencies, ensures all required resources are included and ordered correctly, and handles anonymisation where necessary.

# Approach and Design

Our solution, called seed-maker, enables users to configure and prepare content for seeding other instances via both a Command-Line Interface (CLI) and a web application. The CLI offers direct command-based functionality, while the web app provides a user-friendly, form-based interface.

## Features and Process Overview

The workflow includes the following stages:

1. **Component Selection**: Users are presented with a list of available components for selection, allowing flexibility in creating custom seeds.
2. **Dependency Resolution**: The tool identifies dependencies among selected components and ensures correct import order and UUID handling.
3. **S3 File Retrieval**: Necessary files from S3 are included in the final package, based on user selections and dependencies.
3. **Packaging and Deployment**: The final seed recipe includes JSON file describing the order of seeding, SQL scripts, and data files organised into a directory structure, packaged as a ZIP file for easy distribution and deployment into the DSW Data Seeder.

![User-Data Seed Maker interactions](/figures/path.jpg)

### Component Selection

User interactions begin by selecting a DSW instance and specifying the desired content components for seeding. If no specific components are selected, all available components are listed. The user receives a JSON file with a summary of available content resources, enabling them to select the content they wish to include in the seed package.

Examples of available content for collection and seeding include:

* Project Importers
* Users
* Knowledge Models
* Document Templates
* Projects
* Documents

Below is an example of a JSON output listing available document resources for example instance containing 2 documents:

```
"documents": [
        {
            "uuid": "0000000000-0000-0000-0000-000000000000",
            "name": "Anna Project Horizon 2020"
        },
	{
            "uuid": "111111111111-1111-1111-1111-111111111111",
            "name": "Ben Project Horizon 2020"
        }
    ]
```

### Dependency Resolution

Dependencies between different content resources are a primary challenge we needed to address. Analysing these dependencies and relationships was essential, as shown in the diagram below. We mapped foreign keys to detect dependencies, which inform several key actions:

1. **Adding content:** Sometimes, content must be included even if the user has not selected it for seeding. As illustrated in the diagram, many resources depend on others; for example, knowledge models may depend on a different instance of the same resource. Users are typically unaware of these dependencies, so they may not realise which content is necessary for functionality. To ensure a seamless experience, we automatically add any dependent resources without requiring user input.

2. **Insertion Order:** The sequence of data insertion is critical. For example, documents depend on both document templates and knowledge models, meaning that documents cannot be inserted without the necessary templates and models in place. Failing to insert in the correct order would cause a foreign key constraint violation in SQL. Thus, the tool ensures that inserts are ordered correctly based on dependencies and resource instances.

Our code handles two main types of dependencies: standard data dependencies and dependencies among supplementary tables related to a "main" resource type. For instance, document templates are represented by multiple tables, with one primary table visible to users and two additional tables that are not visible but are still dependent on document templates. This structure ensures all related data is managed properly.

![Diagram of dependencies](/figures/data_dependencies.jpg)

### S3 Files Retrieval

Some DSW content, such as files or images, is stored in S3 object storage rather than the database. The seed-maker tool secures a connection to the S3 storage, retrieves the files associated with the selected resources, and incorporates them into the package.

The S3 has its own file structure that mostly corresponds to the data model from the PostgreSQL database, as shown in the code below, which maps each resource type to its file path in S3.

```
resource_s3_objects = {
    "users": "",
    "projects": "",
    "documents": "documents/",
    "project_importers": "",
    "knowledge_models": "",
    "locales": "locales/",
    "document_templates": "",
    "document_template_asset": "templates/{placeholder}/",
    "document_template_file": ""
}
```

### Packaging and Deployment

The final seed package, ready for input into the DSW Data Seeder, includes:

* **JSON file** detailing the seeding order
* **SQL scripts** with "INSERT" statements
* **Organised data files** in a directory structure

The package structure ensures correct alignment with dependency rules, facilitating seamless deployment.

![Structure of the content package](/figures/package_content.jpg)


## Implementation Details

Our solution uses PostgreSQL for database interactions and MinIO for S3-compatible object storage. Python was chosen as the primary language for its extensive libraries for database interaction, data manipulation, and API integration.

Key functions include:
- **UUID Management**: Generates and assigns UUIDs as necessary to maintain unique instance identifiers for each component.
- **Dependency Resolution**: Ensures components with dependencies (e.g., knowledge models that rely on user definitions) are included in the correct sequence.
- **Anonymisation and Data Transformation**: For questionnaire data, sensitive user information can be anonymised, and unique identifiers reassigned.

This modular structure allows support for diverse DSW configurations, enabling easy export from one instance and import into another.

## User Interface

The seed-maker tool offers both a command-line interface and a simplified web app for creating content packages.


List available resources:

```
dsw-seed-maker list --resource-type "resource_type" --output_file "output_file"
```

* resource_type can be any listable item (e.g., users, knowledge models).
* output_file specifies the file to save the resource list.


Create seed package:

```
dsw-seed-maker seed --input "input_file" --output "output_dir"
```

* input_file contains the desired resources.
* output_dir is the directory for the final content package.

The web app provides a simplified interface, allowing users to select resources via checkboxes. Dependencies are automatically managed—selecting an item with dependencies will automatically select all required items as well. Once the user has finished their selection, they simply press the "Submit" button to process the data. When processing is complete, the download of the zipped file containing the content package begins.

# Conclusions and Future Steps

instance anonymiyation
users annonymization - or do we add them? lets do annonym version now and then optionaly add the users etc
correction of SQL
responses extraction
document_templates smt
logic for listing the dependencies only - for the web app

# Acknowledgements

This work was done during the [BioHackathon Europe][biohack-europe] 2024 organized by ELIXIR in November 2024. We thank the organizers and fellow participants. The development and operation of DSW are supported by ELIXIR CZ research infrastructure (MEYS Grant No. LM2023055).

# References

[biohack-europe]: https://biohackathon-europe.org
[ds-wizard]: https://ds-wizard.org
[elixir-tools-platform]: https://elixir-europe.org/platforms/tools
