# Build Your First AI Data Analyst with Claude Code

An AI-powered data analyst built with Claude Code that connects to a Snowflake data warehouse, automatically cleans data, runs analyses, generates charts, and writes business insights — all from natural language prompts.

Built as part of the LinkedIn Learning Skill Sprint by [Sai Bysani](https://www.linkedin.com/in/saibysani18/).

## What This Project Does

You ask a business question in plain English:
```
Which sellers have the highest revenue but the lowest customer reviews?
```

And the AI Data Analyst automatically:
- Connects to your Snowflake database via MCP
- Picks the right agent for the job
- Joins multiple tables using the correct relationships
- Runs the analysis and generates charts
- Delivers a full report with specific numbers and recommendations

No SQL written. No manual coding. Work that takes a human analyst 1-2 weeks, done in minutes.

## How It Works

The system has five components that work together:

- **CLAUDE.md** — Project memory that gets smarter over time as the system discovers things about your data
- **Skills** — Reusable expertise (data model relationships, reporting standards) that agents load automatically
- **Data Quality Agent** — Specialist that audits and cleans data, finds nulls, duplicates, outliers, and inconsistencies
- **Insight Generator Agent** — Specialist that analyzes clean data, generates charts, and writes business insights
- **MCP Connection** — Bridge between Claude Code and your Snowflake database

## Project Structure
```
AI_Data_Analyst/
  CLAUDE.md                         # Project memory
  mcp_template.json                 # MCP config template (add your credentials)
  .claude/
    agents/
      data_quality.md               # Data cleaning specialist
      insight_generator.md          # Analysis specialist
    skills/
      data_model.md                 # Table relationships and join keys
      reporting_style.md            # Chart and report formatting standards
  outputs/
    cleaned_data/                   # Cleaned CSVs go here
    charts/                         # Generated visualizations go here
    reports/                        # Markdown reports go here
```

## Getting Started

### Prerequisites
- VS Code
- Python (Anaconda recommended)
- Node.js
- Claude Code (`npm install -g @anthropic-ai/claude-code`)

### Setup
1. Clone this repo: `git clone https://github.com/SaiBysani/ai-data-analyst.git`
2. Follow the **Setup Guide** (included with the course) to create a Snowflake account, load the Olist dataset, and connect the MCP server
3. Copy `mcp_template.json` into `.claude/mcp.json` and fill in your Snowflake credentials
4. Open the project in VS Code, start Claude Code with `claude`, and verify the connection with `/mcp`

### Usage
Once connected, just ask business questions:
```
Clean my e-commerce data and generate a quality report.
```
```
What are the top 5 product categories by revenue?
```
```
Analyze delivery performance and its impact on review scores.
```
```
Clean my data, analyze it, and give me a final executive report with charts.
```

## Dataset

This project uses the [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — 9 relational tables with ~100K orders from 2016-2018 covering orders, customers, products, sellers, payments, reviews, and geolocation data.

## Course

This project was built as part of the LinkedIn Learning Skill Sprint: **"Build Your First AI Data Analyst with Claude Code"** — a 7-episode course that walks you through building this system from scratch in 15 minutes.
