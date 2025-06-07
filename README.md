
# Azure Data Factory CI/CD Pipeline with Power BI Integration

## 📘 Overview

This repository implements a robust CI/CD pipeline for deploying Azure Data Factory (ADF) artifacts and Power BI reports/semantic models using Azure DevOps. It supports environment-specific deployments and handles both full and incremental loads, with detailed branch policies and environment segregation.

## 🗂 Project Structure

```.
├── data-factory/
│   ├── pipelines/
│   ├── datasets/
│   └── linkedServices/
├── devops/
│   ├── adf-build-job.yml
│   ├── adf-deploy-job.yml
├── .azure-pipelines/
│   └── azure-pipelines.yml
└── README.md
```

## 🚀 CI/CD Pipeline Flow

### Triggering Pipeline

```yaml
trigger:
  branches:
    include:
      - main

pr:
  branches:
    include:
      - main
```

Only commits to the `main` branch or PRs targeting `main` will trigger the pipeline.

### Stages

#### `BUILD`

- Validates and exports ADF ARM templates
- Optional: Exports Power BI `.pbip` files if configured

#### `DEV`

- Deploys ADF artifacts to the **Development** environment
- Uses variable group `DEV` for environment-specific parameters

#### `PROD`

- Deploys only when changes are merged into `main`
- Uses variable group `PROD`
- Includes branch check to prevent accidental non-main deployments:

  ```yaml
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  ```

## 🔄 Incremental Load Support

### Metadata Table

```sql
CREATE TABLE dbo.ETL_Table_Metadata (
  table_name NVARCHAR(100),
  schema_name NVARCHAR(50),
  is_incremental BIT,
  watermark_column NVARCHAR(100),
  watermark_type NVARCHAR(20),
  last_loaded_datetime DATETIME,
  last_loaded_integer INT
)
```

Supports both `datetime` and `int` watermarking strategies.

### Copy Activity Logic (Dynamic SQL Query)

```expression
@{if(
  and(equals(item().isIncremental, 1), not(empty(activity('Get last watermark').output.firstRow.watermark_type))),
  if(
    equals(activity('Get last watermark').output.firstRow.watermark_type, 'int'),
    concat(
      'SELECT * FROM ', item().schemaName, '.', item().tableName,
      ' WHERE ', item().watermarkColumn, ' > ',
      activity('Get last watermark').output.firstRow.last_loaded_int
    ),
    concat(
      'SELECT * FROM ', item().schemaName, '.', item().tableName,
      ' WHERE ', item().watermarkColumn, ' > ''',
      activity('Get last watermark').output.firstRow.last_loaded, ''''
    )
  ),
  concat('SELECT * FROM ', item().schemaName, '.', item().tableName)
)}
```

### Stored Procedure for Watermark Update

```sql
CREATE PROCEDURE [dbo].[usp_UpdateWatermark]
    @tableName NVARCHAR(255),
    @newWatermark NVARCHAR(50),
    @watermarkType NVARCHAR(20)
AS
BEGIN
    IF @watermarkType = 'datetime'
    BEGIN
        UPDATE dbo.ETL_Table_Metadata
        SET last_loaded = CONVERT(DATETIME, @newWatermark)
        WHERE table_name = @tableName
    END
    ELSE
    BEGIN
        UPDATE dbo.ETL_Table_Metadata
        SET last_loaded_int = CONVERT(INT, @newWatermark)
        WHERE table_name = @tableName
    END
END
```

## 🛡 Branch Protection Strategy

> Enforced via Azure DevOps UI (not YAML)

1. Navigate to **Repos → Branches → `main` → Branch Policies**
2. Enable:
   - Require pull request approval
   - Prevent direct pushes
   - Enable build validation
   - Optional: Reset reviewer votes on changes

This guarantees:

- All deployments to PROD go through `main`
- Only reviewed and validated code reaches production

## 💡 Notes

- All parameters for linked services and datasets are overridden using pipeline-level default values.
- Folder structure for blob outputs uses timestamped folder partitions:

  ```sql
  /server/database/table/Year=2025/Month=06/Day=06/Hour=15/
  ```

- File naming follows convention:

  ```expression
  schema_table_yyyyMMddHHmmss.parquet
  ```

## ✨ Future Enhancements

- Extend to handle API-based data ingestion in parallel branches
- Power BI .pbip deployment via separate YAML pipeline their own folder
- Add Purview metadata tagging for lineage tracking

## 👨‍💻 Maintainer

**Tunde Morakinyo**  
BI Developer, Azure Data Platform Architect
