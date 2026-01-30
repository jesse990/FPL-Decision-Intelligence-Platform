# FPL-Decision-Intelligence-Platform
End to end analytics solution combining Python ETL, Power BI visualization, and DAX-powered predictive modelling for Fantasy Premier League optimization
**Status:** 🚧 Active Development
- **Data Pipeline:** Python (pandas, requests)
- **Visualization:** Power BI Desktop
- **Analysis:** DAX, Power Query (M)
- **Data Source:** FPL Official API + (https://github.com/vaastav/Fantasy-Premier-League)
- **Live Dashboard**: [View on Power BI](https://app.powerbi.com/view?r=eyJrIjoiMzAxYTMzZDctNDMzMy00YjJiLWFkZTAtMTY0MGI3YjYwNWRiIiwidCI6ImQxMjA2OTQzLWJmY2MtNGM3NC04MmQ0LTA1ZTYzYTQzMzViZiJ9)
## Project Motivations
- FPL managers often require multiple different sources in order to acquire all the relevant data they need to make well informed data driven decisions on team selection on a weekly basis. This project aims to be a centralised hub where managers can get all information they need to plan transfers via an interactive easy to use dashboard.

  
!<img src="AdobeExpress-fplpart1.gif" alt="Dashboard Walkthrough" width="800">

!<img src="AdobeExpress-fplpart2.gif" alt="Dashboard Walkthrough" width="800">

## Key Features
- Developed captain recommendation algorithm incorporating weighted performance metrics
- Built validation framework to track recommendation accuracy against actual gameweek outcomes
- Enabled retrospective analysis of prediction patterns and model performance
- Dynamic fixture difficulty analysis for strategic planning
- Player comparison tools

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

**Areas For Improvements**
- Push power query transformations as far back upstream as possible / potentially rework the entire datamodel. Ideally I would like to pull historical data from the api on a  player level using python, then using a direct script via an api web connection in powerquery for updating current data. Some of the problems I anticipate to encounter with this approach is standardising player ids. The api has an ‘element’ identifier but this is season specific, if there is a global player id / code available that would make this possible however its difficult to determine as there's no documentation but i will explore this further.
- This initially started as a learning project but developed more and more over time. If i was to redo i would outline the requirements as early as possible and build everything around those requirements.

- Separate bigger measures into smaller ones. This would allow for a more efficient data model and thus simpler measures

