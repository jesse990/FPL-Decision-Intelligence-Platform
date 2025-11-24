- **GitHub Repository**: [FPL Decision Intelligence Platform](https://github.com/JesseOnu/FPL-Decision-Intelligence-Platform)
- **Live Dashboard**: [View on Power BI](https://app.powerbi.com/view?r=eyJrIjoiMzAxYTMzZDctNDMzMy00YjJiLWFkZTAtMTY0MGI3YjYwNWRiIiwidCI6ImQxMjA2OTQzLWJmY2MtNGM3NC04MmQ0LTA1ZTYzYTQzMzViZiJ9)
# FPL-Decision-Intelligence-Platform
End to end analytics solution combining Python ETL, Power BI visualization, and DAX-powered predictive modelling for Fantasy Premier League optimization
**Status:** 🚧 Active Development
- **Data Pipeline:** Python (pandas, requests)
- **Visualization:** Power BI Desktop
- **Analysis:** DAX, Power Query (M)
- **Data Source:** FPL Official API + (https://github.com/vaastav/Fantasy-Premier-League)

## Key Features
- Developed captain recommendation algorithm incorporating weighted performance metrics
- Built validation framework to track recommendation accuracy against actual gameweek outcomes
- Enabled retrospective analysis of prediction patterns and model performance

## Key Challenges Solved

**Multi-Season Data Integration**
- Identified data quality issues in pre-merged GitHub data
- Built Python pipeline to extract and append 114 individual gameweek files into 3 season-specific datasets
- Validated data integrity through spot-checking against official FPL sources

**Cross-Season Team ID Standardization**  
- Challenge: Team IDs change each season due to promotions/relegations, breaking historical analysis
- Solution: Created a global team identifier system using club 'codes' as a bridge
- Merged fact table with season-specific dimension tables, then consolidated 3 ID columns into one standardized team key
- Result: Seamless cross-season analysis with proper relationships maintained

**Data Source Continuity**
- Original GitHub repository stopped updating at Gameweek 6
- Leveraged AI tools to rapidly develop Python script for direct FPL API extraction  
- Implemented automated weekly data refresh process to maintain current data
