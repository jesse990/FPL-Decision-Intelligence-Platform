# Architecture Overview

## Project Purpose

The FPL Decision Intelligence Platform is an end-to-end analytics solution that processes Fantasy Premier League data and delivers actionable insights through interactive Power BI visualizations. The system ingests data from the official FPL API, applies advanced feature engineering and validation logic, and surfaces predictive recommendations particularly for captaincy selection optimization.

## Data Flow

### Stage 1: Historical Data (Gameweeks 1-5/6)

Data for the first 5-6 gameweeks of each season comes from the [vaastav Fantasy-Premier-League GitHub repository](https://github.com/vaastav/Fantasy-Premier-League). Initially, I used the pre-merged `merged_gw.csv` files, but discovered significant data quality issues particularly empty fields after gameweek 20. To address this, I built a Python script (`FootballFPLFinal.py`) that extracts individual gameweek files from the repository and `mergingGameweeks.py` to combine them seasonally based on filename prefixes, ensuring data integrity across all three seasons.

### Stage 2: Live Data (Gameweeks 6+)

Around gameweek 5-6, the GitHub repository stopped updating. Since I needed current data for development and testing, I leveraged the official FPL API to fetch live gameweek data in a format matching the historical dataset structure. The `updatedfplscrape.py` script handles this extraction, ensuring consistency between historical and live data.

### Stage 3: Data Validation and Merge

New gameweek data is merged with existing season files using `run_to_manually_update_gw.py`. I implement a cautious validation approach: save the merged file under a new name, validate the output, then replace the original only after confirming integrity. This prevents accidental data loss during updates.

## Key Challenges & Solutions

### Challenge 1: Multi-Season Data Integration

**Problem**

The GitHub repository's pre-merged `merged_gw.csv` files contained significant data quality issues. After gameweek 20 in each season, fields were consistently empty, making the entire second half of each season unusable for analysis.

**Solution**

Rather than relying on pre-merged files, I built a modular Python pipeline that:
- Extracts individual gameweek files directly from the GitHub repository (`FootballFPLFinal.py`)
- Parses each file and validates data completeness
- Groups gameweeks by season using filename prefix patterns (`mergingGameweeks.py`)
- Combines them into clean, season-specific datasets

**Result**

Complete, validated datasets for all three seasons with no missing data. The modular approach also makes it easy to add future seasons without refactoring.

### Challenge 2: Cross-Season Team ID Standardization

**Problem**

The GitHub repository contained both a global team dimension table and season-specific team ID mappings. When I appended all three seasons' gameweek data into a single fact table, I discovered that opponent IDs were season-specific—meaning the same team could have different IDs across seasons due to promotions and relegations. This broke the ability to perform historical analysis across seasons.

**Solution**

I implemented a multi-step standardization approach:
1. Created the main fact table by appending all three seasonal gameweek tables
2. Identified that opponent IDs needed to be mapped to a consistent global identifier
3. Built a merge sequence that:
   - Merged each season's team dimension table with the fact table on `season` and `team_id`
   - Extracted the global `team_code` from the season-specific dimension tables
   - Applied this mapping to create consistent opponent ID columns across all three seasons
   - Consolidated the three opponent columns into a single, standardized column

**Result**

The fact table now maintains proper relationships across seasons, enabling seamless cross-season analysis and historical player comparisons without ID collisions.

### Challenge 3: Data Source Continuity

**Problem**

Mid-development, the GitHub repository stopped receiving updates around gameweek 5-6. I needed current data for ongoing development and personal use, but had no guarantee the repository would resume updates. This created a critical dependency risk.

**Solution**

I built an automated data refresh mechanism using the official FPL API (`updatedfplscrape.py`) that:
- Extracts gameweek data directly from the FPL API
- Formats the output to match the historical GitHub data structure
- Merges seamlessly with existing season files using `run_to_manually_update_gw.py`
- Implements a safe validation workflow: new data is saved under a temporary name, validated for integrity, then replaces the original file only after confirmation

**Result**

The pipeline now operates independently of external repository updates. Data is current, validated, and maintains format consistency with historical data.

## Technical Stack

- **Data Pipeline**: Python (pandas, requests, GitHub API integration)
- **Data Processing**: Custom ETL scripts for extraction, transformation, and validation
- **Visualization**: Power BI Desktop
- **Analytics Layer**: DAX & M (Power Query)
- **Data Sources**: FPL Official API, vaastav Fantasy-Premier-League GitHub repository

## Key Features

- **Captain Recommendation Algorithm**: Weighted performance metrics incorporating form, fixture difficulty, and historical performance
- **Opponent Difficulty Scoring**: Dynamic opponent strength assessment based on defensive statistics and recent form
- **Cross-Season Analysis**: Standardized player comparison and performance tracking across multiple seasons
- **Data Validation Framework**: Automated quality checks and audit trails for all data updates
