<p align="center">
    <a alt="License"
        href="https://github.com/fivetran/dbt_linkedin/blob/main/LICENSE">
        <img src="https://img.shields.io/badge/License-Apache%202.0-blue.svg" /></a>
    <a alt="dbt-core">
        <img src="https://img.shields.io/badge/dbt_Core™_version->=1.0.0_<2.0.0-orange.svg" /></a>
    <a alt="Maintained?">
        <img src="https://img.shields.io/badge/Maintained%3F-yes-green.svg" /></a>
    <a alt="PRs">
        <img src="https://img.shields.io/badge/Contributions-welcome-blueviolet" /></a>
</p>

# LinkedIn Ad Analytics Transformation dbt Package ([docs](https://fivetran.github.io/dbt_linkedin/))
# 📣 What does this dbt package do?
- Produces modeled tables that leverage Linkedin Ad Analytics data from [Fivetran's connector](https://fivetran.com/docs/applications/linkedin-ads) in the format described by [this ERD](https://fivetran.com/docs/applications/linkedin-ads#schemainformation) and builds off the output of our [Linkedin Ads source package](https://github.com/fivetran/dbt_linkedin_source).
- Enables you to better understand the performance of your ads across varying grains:
  - Providing an account, campaign (ad groups in other ad platforms), campaign group (campaigns in other ad platforms), creative, and utm/url level reports.
- Materializes output models designed to work simultaneously with our [multi-platform Ad Reporting package](https://github.com/fivetran/dbt_ad_reporting).
- Generates a comprehensive data dictionary of your source and modeled Linkedin Ad Analytics data through the [dbt docs site](https://fivetran.github.io/dbt_linkedin/).

The following table provides a detailed list of all models materialized within this package by default. 
> TIP: See more details about these models in the package's [dbt docs site](https://fivetran.github.io/dbt_linkedin/#!/overview?g_v=1&g_e=seeds).

| **Model**                          | **Description**                                                                                                        |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| [linkedin_ads__account_report](https://fivetran.github.io/dbt_linkedin/#!/model/model.linkedin.linkedin_ads__account_report)        | Each record represents the daily ad performance of each account.                                                       |
| [linkedin_ads__campaign_report](https://fivetran.github.io/dbt_linkedin/#!/model/model.linkedin.linkedin_ads__campaign_report)       | Each record represents the daily ad performance of each campaign. Linkedin campaigns map onto ad groups in other ad platforms.                                                      |
| [linkedin_ads__campaign_group_report](https://fivetran.github.io/dbt_linkedin/#!/model/model.linkedin.linkedin_ads__campaign_group_report) | Each record represents the daily ad performance of each campaign group. Linkedin                                                 |
| [linkedin_ads__creative_report](https://fivetran.github.io/dbt_linkedin/#!/model/model.linkedin.linkedin_ads__creative_report) | Each record represents the daily ad performance of each creative.                                                |
| [linkedin_ads__url_report](https://fivetran.github.io/dbt_linkedin/#!/model/model.linkedin.linkedin_ads__url_report) | Each record represents the daily ad performance of each url.                                                |

# 🎯 How do I use the dbt package?
## Step 1: Prerequisites
To use this dbt package, you must have the following:
- At least one Fivetran Linkedin Ad Analytics onnector syncing data into your destination. 
- A **BigQuery**, **Snowflake**, **Redshift**, **PostgreSQL**, or **Databricks** destination.

### Databricks Dispatch Configuration
If you are using a Databricks destination with this package you will need to add the below (or a variation of the below) dispatch configuration within your `dbt_project.yml`. This is required in order for the package to accurately search for macros within the `dbt-labs/spark_utils` then the `dbt-labs/dbt_utils` packages respectively.
```yml
dispatch:
  - macro_namespace: dbt_utils
    search_order: ['spark_utils', 'dbt_utils']
```

## Step 2: Install the package
Include the following Linkedin Ads package version in your `packages.yml` file:
> TIP: Check [dbt Hub](https://hub.getdbt.com/) for the latest installation instructions or [read the dbt docs](https://docs.getdbt.com/docs/package-management) for more information on installing packages
```yml
# packages.yml
packages:
  - package: fivetran/linkedin
    version: [">=0.5.0", "<0.6.0"]
```

## Step 3: Define database and schema variables
By default, this package runs using your destination and the `linkedin_ads` schema. If this is not where your Linkedin Ad Analytics data is (for example, if your Linkedin schema is named `linkedin_ads_fivetran`), add the following configuration to your root `dbt_project.yml` file:

```yml
# dbt_project.yml
vars:
    linkedin_ads_schema: your_schema_name
    linkedin_ads_database: your_destination_name 
```

## (Optional) Step 4: Additional configurations

### Switching to Local Currency
Additionally, the package allows you to select whether you want to add in costs in USD or the local currency of the ad. By default, the package uses USD. If you would like to have costs in the local currency, add the following variable to your `dbt_project.yml` file:

```yml
# dbt_project.yml
vars:
    linkedin_ads__use_local_currency: True # false by default -- uses USD
```

### Passing Through Additional Metrics
By default, this package will select `clicks`, `impressions`, and `cost` from `ad_analytics_by_creative` and `ad_analytics_by_campaign`. If you would like to pass through additional metrics to the end report models, add the variable configuration in the code block below to your `dbt_project.yml` file. These variables allow for the pass-through fields to be aliased (`alias`) and casted (`transform_sql`) if desired, but not required. Datatype casting is configured via a sql snippet within the `transform_sql` key. You may add the desired sql while omitting the `as field_name` at the end and your custom pass-though fields will be casted accordingly. Use the below format for declaring the respective pass-through variables:

```yml
# dbt_project.yml
vars:
    linkedin_ads__campaign_passthrough_metrics: # this will pass through fields to the account, campaign, and campaign group report models.
        - name: "new_custom_field"
          alias: "custom_field"
        - name: "unique_int_field"
          alias: "field_id"
          transform_sql: "cast(field_id as int)"
    linkedin_ads__creative_passthrough_metrics: # this will pass through fields to the creative and url report models.
        - name: "new_custom_field"
          alias: "custom_field"
        - name: "unique_int_field"
          transform_sql: "unique_int_field / 100.0"
```

> Please ensure you use due diligence when adding metrics to these models. The metrics added by default (`clicks`, `impressions`, and `cost`) have been vetting by the Fivetran team maintaining this package for accuracy. There are metrics included within the source reports which are comprised of averages. You will want to ensure whichever metrics you pass through are indeed appropriate to aggregate.

### Changing the Build Schema
By default this package will build the LinkedIn Ad Analytics staging models within a schema titled (<target_schema> + `_linkedin_ads_source`) and the LinkedIn Ad Analytics final models within a schema titled (<target_schema> + `_linkedin_ads`) in your target database. If this is not where you would like your modeled LinkedIn data to be written to, add the following configuration to your `dbt_project.yml` file:

```yml
# dbt_project.yml
models:
    linkedin:
      +schema: my_new_schema_name # leave blank for just the target_schema
    linkedin_source:
      +schema: my_new_schema_name # leave blank for just the target_schema
```

### Change the source table references
If an individual source table has a different name than the package expects, add the table name as it appears in your destination to the respective variable:
> IMPORTANT: See this project's [`dbt_project.yml`](https://github.com/fivetran/dbt_linkedin/blob/main/dbt_project.yml) variable declarations to see the expected names.
    
```yml
# dbt_project.yml
vars:
    linkedin_ads_<default_source_table_name>_identifier: your_table_name 
```

## (Optional) Step 5: Orchestrate your models with Fivetran Transformations for dbt Core™
Fivetran offers the ability for you to orchestrate your dbt project through [Fivetran Transformations for dbt Core™](https://fivetran.com/docs/transformations/dbt). Learn how to set up your project for orchestration through Fivetran in our [Transformations for dbt Core™ setup guides](https://fivetran.com/docs/transformations/dbt#setupguide).

# 🔍 Does this package have dependencies?
This dbt package is dependent on the following dbt packages. Please be aware that these dependencies are installed by default within this package. For more information on the following packages, refer to the [dbt hub](https://hub.getdbt.com/) site.
> IMPORTANT: If you have any of these dependent packages in your own `packages.yml` file, we highly recommend that you remove them from your root `packages.yml` to avoid package version conflicts.
```yml
packages:
    - package: fivetran/linkedin_source
      version: [">=0.5.0", "<0.6.0"]
    - package: fivetran/fivetran_utils
      version: [">=0.3.0", "<0.4.0"]
    - package: dbt-labs/dbt_utils
      version: [">=0.8.0", "<0.9.0"]
```

# 🙌 How is this package maintained and can I contribute?
## Package Maintenance
The Fivetran team maintaining this package _only_ maintains the latest version of the package. We highly recommend you stay consistent with the [latest version](https://hub.getdbt.com/fivetran/linkedin/latest/) of the package and refer to the [CHANGELOG](https://github.com/fivetran/dbt_linkedin/blob/main/CHANGELOG.md) and release notes for more information on changes across versions.

## Contributions
A small team of analytics engineers at Fivetran develops these dbt packages. However, the packages are made better by community contributions!

# 🏪 Are there any resources available?
- If you have questions or want to reach out for help, please refer to the [GitHub Issue](https://github.com/fivetran/dbt_linkedin/issues/new/choose) section to find the right avenue of support for you.
- If you would like to provide feedback to the dbt package team at Fivetran or would like to request a new dbt package, fill out our [Feedback Form](https://www.surveymonkey.com/r/DQ7K7WW).
- Have questions or want to just say hi? Book a time during our office hours [on Calendly](https://calendly.com/fivetran-solutions-team/fivetran-solutions-team-office-hours) or email us at solutions@fivetran.com.