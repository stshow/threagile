# Threagile POC: OWASP Juice Shop

This repository demonstrates the use of **Threagile**, an open-source tool for "Agile Threat Modeling." It allows you to define your architecture in a YAML file and automatically generate threat models, data flow diagrams, and risk reports.

## Overview

The goal of this POC is to model the architecture of the **OWASP Juice Shop** and identify potential security risks early in the development lifecycle using a "security-as-code" approach.

## 1. The Input Model (`juice-shop.yaml`)

The core of Threagile is the model file. In this directory, `juice-shop.yaml` defines:

- **Information Assets:** Data like `user-credentials`.
- **Trust Boundaries:** Network segments such as `public-internet` and `internal-network`.
- **Technical Assets:** Components like the `angular-frontend`, `node-api`, and `sqlite-db`.
- **Communication Links:** How data flows between assets (e.g., HTTPS from browser to frontend).

Review the file to see how security properties (like encryption, authentication, and authorization) are declared.

## 2. Running Threagile

To process the model and generate the security artifacts, the following command is typically used:

```bash
threagile -model juice-shop.yaml -output .
```

This command parses the YAML, applies built-in risk rules, and renders the output files.

## 3. Reviewing the Results

After execution, Threagile generates several key artifacts found in this directory:

### Visual Diagrams
- **`data-flow-diagram.png`**: An automatically generated DFD showing trust boundaries and communication paths.
  ![Data Flow Diagram](data-flow-diagram.png)
- **`data-asset-diagram.png`**: Visualizes which assets process or store specific data types.
  ![Data Asset Diagram](data-asset-diagram.png)

### Comprehensive Reports
- **`report.pdf`**: A high-level executive summary and detailed technical breakdown of all identified risks.
- **`risks.json` / `risks.xlsx`**: A structured list of security risks categorized by severity (Critical, High, Medium, Low).

### Statistics and Metadata
- **`stats.json`**: Quantitative data about the threat model (e.g., number of assets, links, and risks).
- **`tags.xlsx`**: A summary of tags used across assets for categorization and reporting.
- **`technical-assets.json`**: A machine-readable export of the modeled architecture.

## 4. Key Findings

By analyzing the `risks.xlsx` or the `report.pdf`, you can identify common architectural flaws such as:
- Unencrypted communication across trust boundaries.
- Missing authentication on critical internal APIs.
- Potential SQL injection points based on the technology stack (e.g., SQLite).

## Summary

Threagile transforms manual threat modeling into a repeatable, automated process that fits into a modern CI/CD pipeline. By maintaining the `juice-shop.yaml` file alongside the source code, security becomes a primary citizen of the development workflow.

## Workflows to Investigate
**Workflow 1: LLM as YAML Generator**
IaC/Diagram/Prose → LLM Prompt → `threagile.yaml` → Human Review → `docker run threagile` → Risk Report

The most practical workflow in the wild right now is using an LLM (GPT-4, Claude, Gemini) to generate the threagile.yaml from existing artifacts:

- Feed it: Terraform/CloudFormation code, architecture diagrams, prose descriptions, draw.io exports, Mermaid diagrams
- Prompt it to produce a valid Threagile YAML stub using the schema
- Human reviews/fixes, then runs docker run threagile/threagile

**Workflow 2: Agentic YAML Generation**
IaC/Diagram Ingest → Asset Extraction → Trust Boundary Detection → Data Flow Mapping → YAML Synthesis → `threagile` Run → Risk Report Parse → Finding Triage/Ticket Creation

AWS documented a LangGraph-based agentic architecture that decomposes threat modeling into discrete specialized steps: image processing → asset identification → data flow analysis → threat enumeration, with each node producing standardized outputs feeding the next stage. 

This same pattern maps cleanly onto Threagile:
```
Agent Graph:
[IaC/Diagram Ingest] → [Asset Extraction] → [Trust Boundary Detection] 
→ [Data Flow Mapping] → [Threagile YAML Synthesis] → [threagile run] 
→ [Risk Report Parse] → [Finding Triage/Ticket Creation]
```

[Read More](https://aws.amazon.com/blogs/machine-learning/accelerate-threat-modeling-with-generative-ai/)


**Workflow 3: [STRIDE-GPT](https://github.com/mrwadams/stride-gpt) Complementary Layer**
Prose/Diagram → STRIDE-GPT (first-pass STRIDE) → Translate Findings → `threagile.yaml` → CI/CD Tracking

**Workflow 4: LLM-Assisted Custom Risk Rules**
Natural Language Policy → LLM → Go Rule Code → Threagile `.so` Plugin → Extended Ruleset

Threagile supports custom risk rules as `.so` plugins (Go). Pattern: use an LLM to generate Go rule code from a natural language policy description. 

Example:
```
Write a Threagile custom risk rule in Go that flags any technical asset 
with authentication: none AND internet-facing: true as CRITICAL.
```

Result (It might not work, just a quick Gemini 3 AI-gen example using the above prompt):

```
package main

import (
	"fmt"
	"://github.com"
)

type UnauthenticatedInternetFacingRule struct{}

func (r UnauthenticatedInternetFacingRule) Category() model.RiskCategory {
	return model.RiskCategory{
		Id:    "unauthenticated-internet-facing-asset",
		Title: "Unauthenticated Internet-Facing Asset",
		Description: "Technical assets exposed to the internet without authentication " +
			"pose a critical risk of unauthorized access and exploitation.",
		Impact:     "Critical exposure of system functionality and data to any internet user.",
		ASVS:       "V2 - Authentication Verification Requirements",
		CWES:       []int{287, 306}, // Improper Authentication, Missing Authentication for Critical Function
		STRIDE:     model.Spoofing,
		Mitigation: "Implement a strong authentication mechanism (e.g., OIDC, MFA) " +
			"or move the asset behind a secure VPN/Gateway.",
		Check:          "Check technical asset authentication and internet-facing properties.",
		Function:       model.Operations,
		STRIDEExploits: []model.STRIDE{model.Spoofing, model.InformationDisclosure},
	}
}

func (r UnauthenticatedInternetFacingRule) SupportedTags() []string {
	return []string{}
}

func (r UnauthenticatedInternetFacingRule) GenerateRisks(parsedModel *model.ParsedModel) []model.Risk {
	risks := make([]model.Risk, 0)
	for _, asset := range parsedModel.TechnicalAssets {
		// Rule Logic: Flag if Internet-Facing is true AND Authentication is None
		if asset.InternetFacing && asset.Authentication == model.None {
			risks = append(risks, model.Risk{
				Category:       r.Category(),
				Severity:       model.Critical, // Explicitly set as CRITICAL
				ExploitationLikelihood: model.VeryLikely,
				ExploitationImpact:     model.ExtremelyHigh,
				DataBreachProbability:  model.Probable,
				AffectedId:     asset.Id,
				MostRelevantDataAssetId: asset.HighestSensitivityDataAssetId(parsedModel),
			})
		}
	}
	return risks
}
```

**Workflow 5: CI/CD Integration**
Code Commit → GitHub Action (Threagile) → JSON Risk Output → LLM Triage Script → Jira/Tickets

```yaml .github/workflows/threat-model.yml
- name: Run Threagile
  uses: Threagile/run-threagile-action@v1
  with:
    model: threagile.yaml

- name: LLM Risk Triage
  run: |
    # Parse JSON output, feed to LLM for prioritization
    python scripts/triage_risks.py output/risks.json
```
The JSON output Threagile generates is clean and LLM-consumable for downstream summarization or Jira/ticket creation.
