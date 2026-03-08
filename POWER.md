---
name: "aws-pricing-calculator"
displayName: "AWS Pricing Calculator"
description: "Generate shareable AWS Pricing Calculator URLs from architecture descriptions, blog posts, or solution documents. Discovers service schemas, looks up real pricing, builds estimate JSON, and POSTs to the Save API."
keywords: ["AWS cost estimate", "pricing calculator", "AWS calculator URL", "cost this architecture", "estimate for blog", "AWS pricing", "calculate AWS costs"]
author: "Eleftherios Chaniotakis"
---

# AWS Pricing Calculator Power

Generate a shareable AWS Pricing Calculator URL from any architecture description.

## Prerequisites

| Prerequisite | Required For | Notes |
|---|---|---|
| [uv](https://docs.astral.sh/uv/) | MCP server | Python package manager - install once globally |
| [AWS Pricing MCP Server](https://github.com/awslabs/mcp/tree/main/src/aws-pricing-mcp-server) | Price lookups | Optional: `get_pricing` tool for real price lookups |

## Setup

After installing this power:

1. **Install uv** (one-time setup):
   
   ```bash
   # macOS/Linux
   curl -LsSf https://astral.sh/uv/install.sh | sh
   
   # Windows
   powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
   
   # Or via pip
   pip install uv
   ```

2. **Verify uv is installed:**
   ```bash
   uvx --version
   ```

3. **Test the MCP server** (optional):
   ```bash
   uvx aws-pricing-calculator-mcp
   ```
   
   Press Ctrl+C to stop. The server will be automatically started by Kiro when needed.

4. **Install AWS Pricing MCP Server** (optional, for automatic price lookups):
   - Open Kiro's MCP configuration
   - Add the AWS Pricing MCP server
   - See [AWS Pricing MCP documentation](https://github.com/awslabs/mcp/tree/main/src/aws-pricing-mcp-server)

The MCP server will be automatically downloaded and run by `uvx` when you use this power!

## Trigger Terms

| Phrase | Example |
|--------|---------|
| AWS cost estimate | "Give me an AWS cost estimate for this architecture" |
| pricing calculator | "Create a pricing calculator for this blog post" |
| calculator URL | "Generate a calculator URL for these services" |
| cost this architecture | "Cost this architecture on AWS" |
| estimate for blog | "Create an estimate for this AWS blog post" |

## Available MCP Tools

This power provides the following MCP tools:

### discover_services
Discover AWS Pricing Calculator service schemas. Fetches live service definitions and extracts configurable components.

**Parameters:**
- `service_codes` (optional): Array of service codes to discover (e.g., `["ec2Enhancement", "amazonS3"]`). Leave empty to list all 430+ available services.

**Returns:** Service schemas with version, template ID, and configurable components.

### build_estimate
Build AWS Pricing Calculator estimate JSON from a specification.

**Parameters:**
- `spec` (required): Estimate specification with groups and services (see format below)

**Returns:** Complete estimate JSON with calculated totals, ready for saving.

### save_estimate
Save an estimate to AWS Pricing Calculator and get a shareable URL.

**Parameters:**
- `estimate` (required): Complete estimate JSON (output from `build_estimate`)

**Returns:** Shareable calculator URL and summary.

### get_region_name
Convert AWS region codes to display names.

**Parameters:**
- `region_code` (required): AWS region code (e.g., `"us-east-1"`)

**Returns:** Display name (e.g., `"US East (N. Virginia)"`)

## Workflow

Follow these 6 steps in order:

### Agent Execution Notes

**When executing this workflow:**
- Use the MCP tools provided by this power (discover_services, build_estimate, save_estimate)
- Tools handle all CloudFront API calls and JSON generation automatically
- Always check tool responses for `success: true` before proceeding
- Use AWS Pricing MCP's `get_pricing` tool for price lookups (don't hardcode prices)

### Step 1: Extract Services

Read the architecture document, blog post, or user description. Identify:
- Each AWS service used
- Instance types, storage sizes, throughput, node counts
- Region (default: us-east-1)
- Grouping (environments, tiers, components)

### Step 2: Discover Schemas

For each service, use the `discover_services` tool to get the current schema:

```
discover_services(service_codes=["ec2Enhancement", "amazonS3"])
```

This fetches the live service definition from CloudFront and extracts all
configurable `calculationComponents` with their IDs, types, options, and defaults.

**Check `steering/service_formats.md` first** — if the service has a proven
format there, use it directly instead of discovering from scratch.

**To list all available services:**
```
discover_services()
```

### Step 3: Look Up Pricing

Use the AWS Pricing API MCP tool (`get_pricing`) to get current prices:

```
get_pricing(serviceCode, region, filters)
```

Calculate monthly costs: `unit_price * quantity * hours_per_month` (730 hrs).

### Step 4: Build Estimate

Create a specification object defining groups and services, then use the `build_estimate` tool:

```
build_estimate(spec={
  "name": "My Estimate",
  "groups": [
    {
      "name": "Production",
      "services": [
        {
          "serviceCode": "ec2Enhancement",
          "serviceName": "Amazon EC2",
          "estimateFor": "template",
          "version": "0.0.68",
          "region": "us-east-1",
          "monthlyCost": 175.20,
          "configSummary": "1x m5.xlarge Linux On-Demand",
          "calculationComponents": { ... }
        }
      ]
    }
  ]
})
```

The tool returns the complete estimate JSON with all required fields populated.

### Step 5: Save to API

Use the `save_estimate` tool to upload the estimate and get a shareable URL:

```
save_estimate(estimate=<estimate_from_build_step>)
```

The tool returns:
- `saved_key`: Unique identifier for the estimate
- `calculator_url`: Shareable URL to open in browser
- `summary`: Cost breakdown (monthly, annual)

### Step 6: Return Results

Provide the user with:
1. The calculator URL
2. A cost summary table (service, monthly, annual)
3. Total monthly and annual costs

## MCP Tools Reference

### Tool: discover_services

**Purpose:** Fetch service schemas from AWS Pricing Calculator

**Input Schema:**
```json
{
  "service_codes": ["ec2Enhancement", "amazonS3"]  // Optional, omit to list all
}
```

**Output:**
```json
{
  "success": true,
  "schemas": {
    "ec2Enhancement": {
      "serviceName": "Amazon EC2",
      "serviceCode": "ec2Enhancement",
      "templateId": "template",
      "version": "0.0.68",
      "components": [...]
    }
  }
}
```

### Tool: build_estimate

**Purpose:** Build complete estimate JSON from specification

**Input Schema:**
```json
{
  "spec": {
    "name": "My Estimate",
    "groups": [
      {
        "name": "Production",
        "services": [
          {
            "serviceCode": "ec2Enhancement",
            "serviceName": "Amazon EC2",
            "estimateFor": "template",
            "version": "0.0.68",
            "region": "us-east-1",
            "monthlyCost": 175.20,
            "configSummary": "1x m5.xlarge Linux",
            "calculationComponents": {...}
          }
        ]
      }
    ]
  }
}
```

**Output:**
```json
{
  "success": true,
  "estimate": {...},
  "summary": {
    "name": "My Estimate",
    "groups": 1,
    "services": 1,
    "monthly_cost": 175.20,
    "annual_cost": 2102.40
  }
}
```

### Tool: save_estimate

**Purpose:** Save estimate to AWS and get shareable URL

**Input Schema:**
```json
{
  "estimate": {...}  // Output from build_estimate
}
```

**Output:**
```json
{
  "success": true,
  "saved_key": "abc123def456",
  "calculator_url": "https://calculator.aws/#/estimate?id=abc123def456",
  "summary": {
    "name": "My Estimate",
    "services": 1,
    "monthly_cost": 175.20,
    "annual_cost": 2102.40
  }
}
```

### Tool: get_region_name

**Purpose:** Convert region code to display name

**Input Schema:**
```json
{
  "region_code": "us-east-1"
}
```

**Output:**
```json
{
  "success": true,
  "region_code": "us-east-1",
  "region_name": "US East (N. Virginia)"
}
```

## Tools

## Python Scripts (Internal)

The following Python scripts power the MCP server (users don't interact with these directly):

| Script | Purpose |
|--------|---------|
| `scripts/calc_utils.py` | Shared: curl helpers, UUID, constants, region map |
| `scripts/calc_discover.py` | Fetch service defs, extract configurable components |
| `scripts/calc_build.py` | Build estimate JSON from a spec file |
| `scripts/calc_save.py` | POST to Save API, return shareable URL |
| `mcp_server/server.py` | MCP server entry point |
| `mcp_server/tools.py` | MCP tool implementations |

## Critical Rules

1. **`calculationComponents` must NOT be empty** — populate all required fields or the calculator shows $0.00
2. **`serviceCost.monthly` must be pre-calculated** — the calculator displays this value as-is on load; it does not recalculate
3. **`estimateFor` must match the template ID** from the service definition (e.g., `"template"` for EC2, `"appStream2"` for AppStream)
4. **`version` must match the current service definition version** — always fetch fresh via `calc_discover.py`
5. **Use `curl` subprocess, not Python `requests`** — Python 3.14 has SSL issues with CloudFront CDN endpoints
6. **Use AWS Pricing API MCP** (`get_pricing`) for real price lookups — never hardcode prices
7. **Check `references/service_formats.md`** for known working formats before building from scratch

## References

| File | Contents |
|------|----------|
| `steering/api_endpoints.md` | CloudFront URLs, Save API request/response format |
| `steering/service_formats.md` | Proven `calculationComponents` for 9+ services |
| `steering/troubleshooting.md` | Common issues: $0 costs, read-only warnings, SSL errors |

## License & Attribution

**License:** [Original License]

**Power Author:** [Eleftherios Chaniotakis]

**Original Work:** This power is inspired by [original-skill](https://github.com/quincysting/aws-pricing-calculator) by [Ian Qin](https://github.com/quincysting).

**Source Version:** Based on [version/commit reference].

**Update Frequency:** This power will be updated periodically.

## Quick Start Examples

### From a blog post URL
```
User: "Create an AWS pricing calculator for this blog post: https://aws.amazon.com/blogs/big-data/..."
```
1. Fetch and read the blog post
2. Extract architecture: services, instance types, storage, throughput
3. Use `discover_services` to get schemas for identified services
4. Use AWS Pricing MCP `get_pricing` to look up costs
5. Use `build_estimate` to create the estimate JSON
6. Use `save_estimate` to get the shareable URL

### From a solution document
```
User: "Cost this architecture: 3 EC2 m5.xlarge Windows, 1 RDS Oracle db.m5.xlarge, EBS io1 1400GB"
```
1. Parse the service specs from the description
2. Use `discover_services(service_codes=["ec2Enhancement", "amazonRdsForOracle", "amazonElasticBlockStore"])`
3. Look up pricing with AWS Pricing MCP
4. Build spec with calculationComponents from steering/service_formats.md
5. Use `build_estimate` then `save_estimate`
6. Return URL and cost summary

### From a CSV/spreadsheet
```
User: "Build a calculator estimate from this CSV with our environment sizing"
```
1. Read the CSV to extract services per environment
2. Use `discover_services` for each unique service
3. Map to service codes and calculationComponents
4. Use `build_estimate` with grouped services
5. Use `save_estimate` to get URL
