**Threagile POC Guide: Agile Threat Modeling for OWASP Juice Shop (Docker-only)**

This guide gives you a **complete, copy-paste-ready POC** to spin up Threagile + the classic vulnerable OWASP Juice Shop in <10 minutes. You'll model Juice Shop's architecture as a living YAML document, run automated threat analysis (risks, DFDs, data-asset diagrams, PDF report, Excel/JSON exports), and see how changes to the model instantly update everything — perfect for turning threat models into living Git-tracked docs in your team.

**Why this POC proves value for your team**
- **Zero install**: Everything runs in Docker (no Go, no Node deps).
- **Living docs**: YAML lives in Git → CI can re-generate reports on every PR.
- **Juice Shop integration**: You run the real vulnerable app side-by-side and model its real-world architecture (browser → Node.js/Express/Angular app → SQLite + other services).
- Threagile auto-applies 40+ built-in risk rules (unencrypted comms, weak auth, exposed DBs, XSS-prone web apps, etc.) and flags the exact kinds of issues Juice Shop is famous for.

### Prerequisites
- Docker Desktop / Docker Engine running (tested on latest as of Feb 2026).
- Terminal (bash/zsh/PowerShell with `$(pwd)` support).
- Optional but recommended: VS Code + YAML extension (for autocompletion after `--create-editing-support`).

### Step 1: Create POC workspace & launch Juice Shop (the vulnerable target)
```bash
mkdir threagile-juice-poc && cd threagile-juice-poc
# Run Juice Shop in background (classic vulnerable OWASP app)
docker run -d --name juice-shop -p 3000:3000 bkimminich/juice-shop
```
Open http://localhost:3000 in your browser — you now have the real vulnerable app running.

### Step 2: Pull Threagile Docker image & create starter models
```bash
# Pull latest (or pin to :latest / specific tag if you prefer)
docker pull threagile/threagile

# Create full rich example model (best for learning — models a customer portal with web app, DB, etc.)
docker run --rm -it -v "$(pwd)":/app/work threagile/threagile \
  --create-example-model --output /app/work

# (Optional) Also create minimal stub if you want a clean slate
docker run --rm -it -v "$(pwd)":/app/work threagile/threagile \
  --create-stub-model --output /app/work

# (Optional) Generate IDE editing support (schema + live templates for VS Code)
docker run --rm -it -v "$(pwd)":/app/work threagile/threagile \
  --create-editing-support --output /app/work
```

You now have:
- `threagile-example-model.yaml` (recommended starting point — copy/rename to `juice-shop-model.yaml`)
- `threagile-stub-model.yaml`
- `schema.json`, `live-templates.txt`, etc. for your editor.

### Step 3: Customize the model for OWASP Juice Shop
Copy the example and adapt it (or start from stub). Here's a **ready-to-use minimal Juice Shop model** you can paste into `juice-shop-model.yaml`:

```yaml
threagile_version: 1.0.0
title: OWASP Juice Shop - Threat Model POC
date: 2026-02-26
author:
  name: Your Team
  homepage: https://yourcompany.com

business_criticality: important
management_summary_comment: "POC demonstrating living threat model for vulnerable OWASP Juice Shop"

data_assets:
  User Accounts & PII:
    id: user-accounts
    description: Customer accounts, hashed passwords, PII, order history
    usage: business
    origin: End Users
    owner: Juice Shop
    quantity: many
    confidentiality: strictly-confidential
    integrity: critical
    availability: critical
    justification_cia_rating: "Contains login creds and personal data — classic Juice Shop target"

  Product & Order Data:
    id: product-data
    description: Juice catalog, orders, payments (insecure by design)
    usage: business
    quantity: many
    confidentiality: confidential
    integrity: critical
    availability: important

technical_assets:
  End-User Browser:
    id: browser
    description: Customer browser accessing Juice Shop
    type: external-entity
    usage: business
    used_as_client_by_human: true
    technology: browser
    internet: true
    machine: physical
    owner: End User
    data_assets_processed: [user-accounts, product-data]

  Juice Shop Web Application:
    id: juice-webapp
    description: Node.js + Express + Angular monolithic web app (intentionally vulnerable)
    type: web_application   # or process if you prefer
    usage: business
    size: application
    technology: nodejs     # or web_application, containerized
    machine: container
    internet: true
    owner: Juice Shop Team
    data_assets_processed: [user-accounts, product-data]
    data_assets_stored: [user-accounts, product-data]
    communication_links:
      Frontend to Backend (same process but modeled):
        target: juice-webapp
        description: Internal calls
        protocol: http   # intentionally insecure in Juice Shop
        authentication: session-id
        authorization: enduser-identity-propagation

  SQLite Database:
    id: juice-db
    description: SQLite backend (file-based, no auth in many Juice Shop scenarios)
    type: database
    usage: business
    technology: sqlite
    machine: container
    encryption: none
    owner: Juice Shop Team
    data_assets_stored: [user-accounts, product-data]
    data_assets_processed: [user-accounts, product-data]

  Email / External Services (optional vulnerable sink):
    id: email-service
    description: SMTP for password resets, etc. (often misconfigured)
    type: external-entity
    usage: business
    internet: true
    technology: smtp

trust_boundaries:
  Internet to App:
    id: internet-boundary
    description: Public internet to Juice Shop
    type: network
    technical_assets_inside: [juice-webapp]
```

**Key customizations you just did**:
- Marked `http` (not https) → Threagile will flag "Unencrypted Communication".
- `sqlite` with `encryption: none` → flags DB risks.
- Web app processes/stores sensitive data → triggers injection, XSS, broken auth rules.
- You can add more (Kubernetes, cloud, etc.) as your real architecture evolves.

Save the file as `juice-shop-model.yaml`.

### Step 4: Run the analysis (generates living outputs)
```bash
docker run --rm -it -v "$(pwd)":/app/work threagile/threagile \
  --verbose \
  --model /app/work/juice-shop-model.yaml \
  --output /app/work \
  --generate-report-pdf \
  --generate-data-flow-diagram \
  --generate-data-asset-diagram \
  --generate-risks-excel \
  --generate-risks-json \
  --generate-technical-assets-json
```

**Outputs created in your folder** (these are your living documents):
- `report.pdf` → Full risk report with diagrams, risk levels (low/medium/high/critical), mitigation advice.
- `data-flow.png` + `data-asset.png` → Auto-generated DFDs.
- `risks.xlsx` / `risks.json` → Trackable risks (import into Jira/Excel dashboards).
- `stats.json`, `technical-assets.json` → For CI metrics or dashboards.
- Console shows risk summary (you'll see many medium/high risks because it's Juice Shop!).

Re-run the exact command after any YAML edit → everything updates instantly.

### Step 5: Explore & iterate (make it living)
- Open `report.pdf` — see how Threagile caught Juice-Shop-style issues automatically.
- Edit the YAML (e.g., change `protocol: http` → `https`, add `encryption: data-with-symmetric-shared-key` on DB) → re-run the Docker command.
- **Turn into living doc for your team**:
  ```bash
  git init
  git add juice-shop-model.yaml
  git commit -m "Initial Juice Shop threat model"
  ```
  Add a GitHub Actions / GitLab CI step (see official example repo: https://github.com/Threagile/github-integration-example) that runs the same Docker command on every push/PR and attaches the new `report.pdf` + risks as artifacts.

### Bonus: Server / Web UI mode (optional for team demo)
```bash
docker run -d --name threagile-server -p 8080:8080 \
  --mount type=volume,src=threagile-data,target=/data \
  threagile/threagile -server 8080
```
Then open http://localhost:8080 (or use the public playground at https://run.threagile.io for quick tests).

### Cleanup
```bash
docker stop juice-shop threagile-server 2>/dev/null || true
docker rm juice-shop threagile-server 2>/dev/null || true
```

**Next steps**
1. Commit the YAML to your repo.
2. Add the Docker analysis command to CI.
3. Extend the model with your real microservices, cloud services, auth (Keycloak/OAuth), etc.
4. Use risk-tracking section inside the YAML itself to mark "mitigated" status — it stays in the model forever.
