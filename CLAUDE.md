# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an **Azure Data Factory (ADF)** Git-integrated repository for the **Algalif** factory, deployed in the **North Europe** region. It uses ADF's native Git integration — all resources are stored as ARM-compatible JSON definitions.

## Git / Publish Workflow

- The working branch is `main`; ADF publishes ARM templates to the `adf_publish` branch (configured in `publish_config.json`)
- Changes are authored in `main`, then published to ADF via the ADF Studio "Publish" button, which auto-generates the `adf_publish` branch
- Do **not** manually edit the `adf_publish` branch

## Architecture

The factory copies binary files (trendlog data) from on-premises Compass file servers to Azure Blob Storage (`algalifcompass` storage account, `embla` container).

### Data Flow

```
On-prem file servers (Embla, COMPASS) → Self-hosted IR (AssaIntegratedRuntime) → Azure Blob Storage
```

### Key Resources

| Type | Name | Purpose |
|---|---|---|
| **Pipeline** | `CopyFromCompass2` | Copies trendlog files from Embla/Compass2 to blob, filtered by LastModifiedDate |
| **Pipeline** | `CopyNewFilesByLastModifiedDate` | Generic incremental copy pipeline (same pattern, different default parameters) |
| **Trigger** | `12 Hour Updates` | Runs `CopyFromCompass2` every 12 hours (GMT) |
| **Dataset** | `BinaryDataSourceStore` | Parameterized binary source on the on-prem file server (via `Compass2` linked service) |
| **Dataset** | `BinaryDataDestinationStore` | Parameterized binary destination on Azure Blob (via `AlalifBlobService` linked service) |
| **Linked Service** | `Compass2` | File server connection to `\\Embla\F$\Trend` via self-hosted IR |
| **Linked Service** | `Assa Connection` | File server connection to `\\COMPASS\Trendlog` via self-hosted IR |
| **Linked Service** | `AlalifBlobService` | Azure Blob Storage (`algalifcompass`) |
| **Integration Runtime** | `AssaIntegratedRuntime` | Self-hosted IR running on-premises for file server access |

## Editing Conventions

- Each ADF resource is a single JSON file under its resource-type folder (`pipeline/`, `dataset/`, `linkedService/`, `trigger/`, `integrationRuntime/`, `factory/`)
- Pipeline parameters use ADF expression syntax: `@pipeline().parameters.ParamName` and `@{pipeline().parameters.ParamName}`
- Dataset parameters use: `@dataset().ParamName` and `@concat(...)` expressions
- Credentials are stored as `encryptedCredential` values — never replace these with plaintext secrets
