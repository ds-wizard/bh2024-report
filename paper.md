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

This report provides an overview of our activities and accomplishments related to the creation of reusable RDM (Research Data Management) Planning Environments for trainings and workshops conducted during the ELIXIR BioHackathon Europe 2024. ELIXIR recognizes the critical role of effective data management planning in enabling sustainable and reproducible research outcomes. This effectiveness is achieved through the use of appropriate Data Management Planning tools, such as the Data Stewardship Wizard. The Data Stewardship Wizard is used to conduct various trainings which require instance with data which are different for each training. Goal of this project was to provide easy and effective way to prepare "recipes" for DSW Data Seeder.

# Introduction

Using tools such as Data Stewardship Wizard (DSW, ds-wizard.org) to conduct effective Data Management Planning is not a trivial task. Therefore, ELIXIR acknowledges the need to educate users at all levels—whether they are data stewards responsible for preparing the necessary steps to produce Data Management Plans or researchers tasked with providing project information and generating these plans.

Training sessions play a vital role in equipping users with the necessary skills to effectively utilize these tools. They enhance the quality of Data Management Plans and empower users to confidently handle data management tasks. To support frequent trainings, it is essential to prepare accurate and comprehensive materials, including a tailored instance of the Data Stewardship Wizard. The goal of this project was to create a simple and clear method for preparing these training instances by populating them with appropriate data.

## Data Stewardship Wizard

The Data Stewardship Wizard is a comprehensive platform for Research Data Management (RDM) planning. However, the DSW itself is just a platform. To be functional, it needs to be populated with resources. These resources complement each other and work together to create a Data Management Plan. Additionally, these resources can be shared across DSW instances. To prepare a training instance, it is necessary to import these resources into it:

- **Knowledge Models**: Templates of RDM best practices and guidelines that are essential for each DSW instance.  
- **Document Templates**: Templates used to generate structured outputs, such as PDF plans or reports, derived from RDM plans. These can be linked to a specific Knowledge Model or even a particular version of it.  
- **Questionnaires**: Data structures designed to collect responses based on the Knowledge Model.  

The DSW cannot function without these resources. Their relationships are as follows: A Questionnaire is created using a Knowledge Model. The Document Template is designed to be compatible with a Knowledge Model. Together, the Knowledge Model and Questionnaire form a Project. This Project can then utilize a Document Template, which transforms the content of the Project into a Document—the primary output of the Data Stewardship Wizard. See the following figure:

![Data Stewardship Wizard resources relations](/figures/dsw_diagram.png)

Each of these resources is stored in a relational database and may include assets stored in object storage (e.g., MinIO for S3-compatible storage). However, manually setting up these resources and resolving their interdependencies can be challenging. For example, knowledge models rely on user definitions, document templates, and questionnaire responses to function cohesively. This complexity presents significant challenges when replicating DSW setups, particularly in training environments.

To address this, we also need to introduce additional resources that, while not directly used to generate the Document, may serve as dependencies or play a supporting role in the overall process:

- **Users**: A representation of a person within the system. They can have one of the following roles:
  - **Administrator**: Manages the DSW instance; not relevant for this case.
  - **Data Steward**: Prepares Knowledge Models and Document Templates for Researchers to use.
  - **Researcher**: Answers all questions within a Questionnaire and uses Document Templates to generate their Documents.
- **Locales**: Localizations to support multiple languages. This only affects the DSW UI.

The actual work within DSW is divided in the following way:

![Data Stewardship Wizard areas of responsibility](/figures/dsw_diagram_responsibilities.png)

All these resources and their dependencies need to be transferred between instances to fill a training instance with the data necessary to conduct a training. This approach is useful not only for preparing before a training but also afterward, as there is no need to "clear" an instance of excess data created during a training. Instead, it is possible to simply delete everything and create a new instance. To import data into an instance, there is an existing tool, Data Seeder.

## Data Seeder

[Data Seeder][data-seeder] is a tool for seeding Data Stewardship Wizard data. It is used to insert data into any DSW instance. In order for this to work, it is necessary to provide a recipe. The recipe is a collection of all the previously mentioned resources, which may also have various dependencies. Currently, all of this must be prepared manually.

The goal of this project is to improve this process by creating an application that connects to an example or source instance of DSW, lists all available resources there, and enables the automatic creation of the recipe. This application would also automatically resolve all dependencies and include any dependent resources.

# Problem Statement

The DSW team frequently conducts training and workshop sessions to demonstrate the platform’s various features and how DSW integrates with different roles throughout the research data management lifecycle. For each event, a new DSW instance is created for participants to use during demonstrations and hands-on activities. However, each newly created instance initially lacks any content, making it unsuitable for training or workshop use straight away. To make the instance functional, relevant content must first be imported. This process involves gathering selected content resources from another instance(s) and using the DSW’s Data Seeder to populate the instance.

![Using Data Seeder](/figures/seed_maker_diagram.jpg)

Preparing data for the Data Seeder, referred to here as a content package, is currently a manual, time-intensive, and resource-heavy process. Different content resources (such as knowledge models, document templates, etc.) are interdependent and may rely on other, often non-visible, resources. Content package creators must ensure that all dependent resources are included and correctly ordered. Content insertion order is critical due to relational dependencies—adding an item with a foreign key dependency on another item that has not yet been inserted will result in an error. Additionally:

* **Identifiers** associated with instances must be altered to avoid conflicts with the original instance.
* **Author** information for items may need to be anonymised for privacy and security reasons.

Given the rising interest in DSW and the growing demand for training, this manual process is becoming increasingly unsustainable.

To address these challenges, we developed a tool, seed-maker, that automates the creation of content packages, ready for use with the DSW Data Seeder in just minutes. This tool resolves dependencies, ensures all required resources are included and ordered correctly, and handles anonymization where necessary.

# Approach and Design

Our solution, called seed-maker, enables users to configure and prepare content for seeding other instances via both a Command-Line Interface (CLI) and a web application. The CLI offers direct command-based functionality, while the web app provides a user-friendly, form-based interface.

## Features and Process Overview

The workflow includes the following stages:

1. **Component Selection**: Users are presented with a list of available components for selection, allowing flexibility in creating custom seeds.
2. **Dependency Resolution**: The tool identifies dependencies among selected components and ensures correct import order and UUID handling.
3. **S3 File Retrieval**: Necessary files from S3 are included in the final package, based on user selections and dependencies.
3. **Packaging and Deployment**: The final seed recipe includes JSON file describing the order of seeding, SQL scripts, and data files organized into a directory structure, packaged as a ZIP file for easy distribution and deployment into the DSW Data Seeder.

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

Dependencies between different content resources are a primary challenge we needed to address. Analyzing these dependencies and relationships was essential, as shown in the diagram below. We mapped foreign keys to detect dependencies, which inform several key actions:

1. **Adding content:** Sometimes, content must be included even if the user has not selected it for seeding. As illustrated in the diagram, many resources depend on others; for example, knowledge models may depend on a different instance of the same resource. Users are typically unaware of these dependencies, so they may not realize which content is necessary for functionality. To ensure a seamless experience, we automatically add any dependent resources without requiring user input.

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
* **Organized data files** in a directory structure

The package structure ensures correct alignment with dependency rules, facilitating seamless deployment.

![Structure of the content package](/figures/package_content.jpg)

## Implementation Details

Our solution uses PostgreSQL for database interactions and MinIO for S3-compatible object storage. Python was chosen as the primary language for its extensive libraries for database interaction, data manipulation, and API integration.

Key functions include:
- **UUID Management**: Generates and assigns UUIDs as necessary to maintain unique instance identifiers for each component.
- **Dependency Resolution**: Ensures components with dependencies (e.g., knowledge models that rely on user definitions) are included in the correct sequence.
- **Anonymization and Data Transformation**: For questionnaire data, sensitive user information can be anonymised, and unique identifiers reassigned.

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

There can be many different combinations and use cases for recipe creation, whether as part of training sessions or outside of them. The Data Seeder, along with the recipes enabled during this BioHackathon project, can also be used to transfer data seamlessly between different instances of the Data Stewardship Wizard. This presents an opportunity for further development of the Data Seed Maker. Several areas could benefit from improvements or additional features, such as various anonymization options for seeded data, more efficient resource extraction, and the ability to view dependencies in the web app.

- **Instance Anonymization**: The ability to remove information identifying the instance from which the data was taken.
- **Users Anonymization**: The ability to remove information about the users who performed various actions with the resources.
- **Responses Extraction**: The ability to extract not only resources, such as Knowledge Models or Document Templates, but also responses from Projects.
- **Logic for Listing Dependencies Only**: The ability to view the dependencies of a specific resource within the web app GUI.

Overall, the work completed during the ELIXIR BioHackathon 2024 lays a strong foundation for the Seed Maker platform, which will be further developed and used in real-world training sessions. This will significantly reduce the tedious and repetitive work that, until now, had to be done before every training session and when moving resources between instances of the Data Stewardship Wizard.

# Acknowledgements

This work was done during the [BioHackathon Europe][biohack-europe] 2024 organized by ELIXIR in November 2024. We thank the organizers and fellow participants. The development and operation of DSW are supported by ELIXIR CZ research infrastructure (MEYS Grant No. LM2023055).

# References

[biohack-europe]: https://biohackathon-europe.org
[data-seeder]: https://github.com/ds-wizard/engine-tools/tree/develop/packages/dsw-data-seeder
[ds-wizard]: https://ds-wizard.org
[elixir-tools-platform]: https://elixir-europe.org/platforms/tools
