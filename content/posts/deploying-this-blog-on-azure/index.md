---
title: "Deploying This Blog on Azure"
date: 2026-09-09
draft: false
author: "Jorge"
description: "How I designed and deployed this blog using Hugo, Azure Static Web Apps, Bicep, and GitHub Actions with Workload Identity Federation. No secrets in the pipeline. No excuses."
tags: ["azure", "bicep", "hugo", "github-actions", "static-web-apps", "wif", "dns", "key-vault"]
categories: ["IaC", "Platform Engineering"]
cover:
  image: "featured.png"
  alt: "Meeple in the Cloud blog architecture diagram"
  relative: true
showToc: true
TocOpen: false
---

A blog about cloud architecture should be deployed with the same rigour you'd apply to any other workload. That means Infrastructure as Code, a proper CI/CD pipeline, zero plaintext secrets, and a cost you can justify — even if you're justifying it only to yourself.

This post walks through every decision I made building **Meeple In The Cloud**: why I chose each component, what I considered and discarded, and the Bicep code that puts it all together.

---

## The constraints I set for myself

Before picking any technology, I defined what I actually needed:

- **Zero-ops** — I want to write, not manage servers or certificates
- **IaC from day one** — the infrastructure must be reproducible from a `git clone`
- **No secrets in the pipeline** — Workload Identity Federation or nothing
- **Custom domain with HTTPS** — `www.meepleinthecloud.com`, managed in Azure
- **Cost under €1/month** — this is a blog, not a SaaS product

These constraints eliminated a lot of options early on.

---

## Architecture overview

```
┌─────────────────────────────────────────────────────┐
│                   GitHub                            │
│                                                     │
│  meeple-in-the-cloud-blog   meeple-in-the-cloud-infra│
│  ├── content/             ├── bicep/                │
│  └── .github/workflows/   └── .github/workflows/   │
│      deploy-blog.yml          deploy-infra.yml      │
└──────────┬──────────────────────────┬───────────────┘
           │ WIF (no secrets)         │ WIF (no secrets)
           ▼                          ▼
┌─────────────────────────────────────────────────────┐
│              Azure Subscription                     │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  rg-mitc-prd-blog                           │   │
│  │                                             │   │
│  │  ┌──────────────┐   ┌───────────────────┐  │   │
│  │  │  DNS Zone    │   │  Key Vault        │  │   │
│  │  │  meeple      │   │  swa-deploy-token │  │   │
│  │  │  inthecloud  │   │  (Secrets Officer)│  │   │
│  │  │  .com        │   │                   │  │   │
│  │  └──────┬───────┘   └────────┬──────────┘  │   │
│  │         │ CNAME www          │ reads token  │   │
│  │         ▼                    ▼              │   │
│  │  ┌──────────────────────────────────────┐  │   │
│  │  │  Azure Static Web Apps (Free)        │  │   │
│  │  │  Hugo build · CDN · HTTPS · Previews │  │   │
│  │  └──────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

Two repos, three Azure resources, two pipelines. That's the whole thing.

---

## The decisions

### Hugo over a CMS

The temptation with a new blog is to reach for something like Ghost or WordPress — a proper CMS with an editor, plugins, and a database. I've been down that road. The operational overhead always ends up eating into writing time.

Hugo generates static HTML from Markdown files. There's no database, no runtime, no process to keep alive. Posts are just files in a git repository — version controlled, diffable, and editable in VS Code. For a technical blog where code blocks are first-class content, Markdown is the right substrate.

The build is also fast enough to feel instantaneous locally, which matters when you're iterating on layout or formatting.

### Azure Static Web Apps over Blob Storage + CDN

Azure Static Web Apps (SWA) is the obvious choice here, but it's worth explaining why over the manual alternative of Blob Storage + Azure CDN.

SWA gives you CDN, HTTPS, and custom domain management in a single resource with zero configuration. The Free tier supports one custom domain, SSL, and up to 100 GB of bandwidth per month — more than enough for a technical blog. The feature that pushed me over the edge is **preview environments**: every pull request gets its own deployment URL automatically. I can review a post before merging it to main, which is the same workflow I'd want on any production workload.

The manual Blob + CDN approach gives you more control over CDN rules and origin configuration, but introduces two resources to manage instead of one, and you lose the PR preview feature. For this use case, the trade-off doesn't make sense.

### Azure DNS Zone

If I'm building the blog on Azure, the DNS should live there too. Having the DNS Zone in the same resource group as the SWA means the infrastructure is self-contained — one `az deployment sub create` and everything exists.

The practical reason is that the CI/CD pipeline can patch the CNAME record programmatically after each infrastructure deployment, so the DNS always reflects the current SWA hostname without manual intervention.

### Key Vault for the SWA deployment token

Static Web Apps uses a deployment token to authenticate push operations from the pipeline. This token is sensitive — anyone who has it can deploy arbitrary content to your site.

The naive approach is to store it as a GitHub Secret. That works, but it means a human has to retrieve the token from the Azure portal, copy it, and paste it into GitHub. It also means the token lives in GitHub's secret store with no audit trail on the Azure side.

My approach: Bicep reads the token directly from the SWA resource during deployment using `listSecrets()` and writes it to Key Vault. The pipeline then reads it from Key Vault using the WIF identity. No human ever touches the token. Rotation is a matter of running the infrastructure pipeline again.

### Workload Identity Federation over Service Principal secrets

This is non-negotiable for me. A Service Principal with a client secret is a credential that expires, needs rotation, and can be leaked in logs, environment variables, or — if someone is having a bad day — a public git commit.

Workload Identity Federation replaces the secret with a trust relationship: GitHub's OIDC provider issues a short-lived token for each workflow run, Azure validates it against a registered federated credential, and the pipeline gets a scoped Azure token that expires when the job ends. No secret. No rotation. No `AZURE_CLIENT_SECRET` in your GitHub Secrets list.

The only values stored in GitHub Secrets are non-sensitive identifiers: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID`. None of these are secrets in the cryptographic sense.

---

## The Bicep structure

The infrastructure is organised as a subscription-scoped main template that orchestrates three modules:

```
bicep/
├── main.bicep           # Subscription scope — creates RG, calls modules
├── modules/
│   ├── dns.bicep        # DNS Zone + CNAME + TXT verification record
│   ├── keyvault.bicep   # Key Vault + RBAC assignment for WIF identity
│   └── swa.bicep        # Static Web App + custom domain + token → KV
└── params/
    └── prd.bicepparam   # Production parameter values
```

### main.bicep — the orchestrator

```bicep
targetScope = 'subscription'

module dnsZone 'modules/dns.bicep' = {
  name: 'deploy-dns'
  scope: rg
  params: {
    domainName: domainName
    tags: tags
  }
}

module keyVault 'modules/keyvault.bicep' = {
  name: 'deploy-keyvault'
  scope: rg
  params: {
    prefix: prefix
    location: location
    githubActionsObjectId: githubActionsObjectId
    tags: tags
  }
}

module staticWebApp 'modules/swa.bicep' = {
  name: 'deploy-swa'
  scope: rg
  params: {
    prefix: prefix
    location: location
    domainName: domainName
    keyVaultName: keyVault.outputs.keyVaultName
    tags: tags
  }
}
```

The dependency chain is implicit: `swa.bicep` references the Key Vault by name, so Bicep resolves the deployment order automatically.

### The token handoff in swa.bicep

The most interesting part of the Bicep is how the SWA deployment token ends up in Key Vault without any manual step:

```bicep
resource swa 'Microsoft.Web/staticSites@2023-01-01' = {
  name: 'swa-${prefix}-blog'
  // ...
}

resource swaTokenSecret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  parent: keyVault
  name: 'swa-deployment-token'
  properties: {
    value: swa.listSecrets().properties.apiKey
  }
}
```

`listSecrets()` is a Bicep runtime function that calls the SWA management API and returns the deployment token. The result is written directly to Key Vault as part of the same deployment. The token never appears in any output, log, or parameter file.

### RBAC in keyvault.bicep

Key Vault uses the RBAC authorisation model (no access policies). The WIF identity gets `Key Vault Secrets Officer` scoped to the vault:

```bicep
var kvSecretsOfficerRoleId = 'b86a8fe4-44ce-4948-aee5-eccb2c155cd7'

resource githubActionsKvRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, githubActionsObjectId, kvSecretsOfficerRoleId)
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      kvSecretsOfficerRoleId
    )
    principalId: githubActionsObjectId
    principalType: 'ServicePrincipal'
  }
}
```

Using `guid()` with deterministic inputs means the role assignment ID is stable across deployments — re-running the pipeline never creates a duplicate assignment.

---

## The pipelines

### Infrastructure pipeline (deploy-infra.yml)

Triggers on changes to `bicep/**` or manually via `workflow_dispatch`. The job sequence is:

1. **Lint** — `az bicep lint` catches syntax errors before anything hits Azure
2. **What-if** — shows exactly what will change before applying
3. **Deploy** — `az deployment sub create` with the param file
4. **Patch CNAME** — updates the DNS record with the actual SWA hostname from the deployment output

The what-if step is not optional ceremony — it's the equivalent of `terraform plan` and has caught real issues before they became real problems.

### Blog pipeline (deploy-blog.yml)

Triggers on changes to `content/**`, `themes/**`, or Hugo config files. The sequence:

1. **Hugo build** — `hugo --minify --gc` with the `extended` version for SCSS support
2. **Azure login** — WIF token exchange, no secrets
3. **Read token** — `az keyvault secret show` retrieves the SWA deployment token
4. **Deploy** — `Azure/static-web-apps-deploy@v1` pushes the built site
5. **PR comment** — posts the preview URL on pull requests

Pull requests get a staging environment automatically. The preview URL is posted as a PR comment, so reviewing a draft post before publishing is a first-class workflow.

---

## Setting up WIF

Before any of this works, you need the Workload Identity Federation trust relationship in place. The `setup-wif.sh` script handles it:

```bash
./scripts/setup-wif.sh <subscription-id> <github-owner> meeple-in-the-cloud-blog
```

It creates a User-Assigned Managed Identity, registers two federated credentials (one for `main` branch pushes, one for pull requests), and assigns `Contributor` on the subscription for Bicep deployments. The Key Vault role assignment is handled by Bicep itself.

The script outputs exactly what to add to GitHub Secrets and what to fill in `prd.bicepparam`. Follow it in order and the first infrastructure deployment will succeed on the first run.

---

## Cost breakdown

| Resource | SKU | Estimated monthly cost |
|---|---|---|
| Azure Static Web Apps | Free | €0.00 |
| Azure DNS Zone | Standard | ~€0.50 |
| Azure Key Vault | Standard | ~€0.05 |
| **Total** | | **~€0.55/month** |

The DNS Zone is the only resource with meaningful cost, and it's priced per zone plus per million queries. A technical blog doesn't generate a million DNS queries in a month.

---

## What I'd do differently

**Private Endpoints are overkill here.** The Key Vault is publicly accessible, which I'd never accept on a workload that handles customer data. For a blog where the most sensitive thing in the vault is a static site deployment token, the operational complexity of a Private Endpoint and a self-hosted runner isn't justified. Know where to draw the line.

**The SWA Free tier has limits.** 100 GB bandwidth, one custom domain, no SLA. If this blog ever grows to the point where those limits matter, migrating to the Standard tier ($9/month) is a one-line change in the Bicep SKU. Starting on Free and moving up is the right call.

---

## The full code

Everything — Bicep modules, pipeline definitions, and the setup script — is in the [meeple-in-the-cloud-infra](https://github.com/YOUR_GITHUB_USERNAME/meeple-in-the-cloud-infra) repository. Fork it, update the parameter file, run the setup script, and you'll have your own blog infrastructure in Azure in under 20 minutes.

The blog content, including this post, lives in [meeple-in-the-cloud-blog](https://github.com/YOUR_GITHUB_USERNAME/meeple-in-the-cloud-blog).

---

*Next up: Landing Zone from scratch — Management Groups, subscription design, and why the decisions you make on day one are the hardest to undo.*
