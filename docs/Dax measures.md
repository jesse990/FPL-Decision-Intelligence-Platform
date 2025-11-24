##Dax Measures

## Best Scorer (Head-to-Head)

### Purpose

This measure identifies which player from your team has scored the most points historically against your upcoming opponent. It helps inform captain selection by showing which players historically perform well against this specific opponent.

### How It Works

**Step 1: Get Current Context**

The measure captures your current team, opponent, and the current season. It also calculates what the gameweek was 5 weeks ago (`targetgw = gameweek - 5`), so you're only looking at recent form rather than the entire season.

**Step 2: Filter Active Players**

It identifies which players from your team have actually played in the last 5 gameweeks (players with minutes > 0). This avoids showing players wo may still be in the database and available to be bought but have transferred to another league

**Step 3: Build Player Summary Table**

Creates a table with all players and their total season points using `SUMMARIZE`. This gives you a clean dataset of player names, points, and which teams they've played against.

**Step 4: Filter to Your Team**

Narrows down to only players from your current team who played in the last 5 gameweeks.

**Step 5: Find Top Scorer vs This Opponent**

Uses `TOPN(1, ...)` to find the player with the most total points against this specific opponent. Returns their name and point total.

**Step 6: Handle Missing Data**

If there's no history (first time these teams play), returns "no history".

### DAX Code
```dax
Best scorer = 
VAR currentteam = CALCULATE(SELECTEDVALUE(futurfixures[teamid]), AllPlayerMergedGameWeek[season] = "2025-26")
VAR currentopp = SELECTEDVALUE(futurfixtures[oppid])
VAR curseason = "2025-26"
VAR gameweek = 
    CALCULATE(MAX(AllPlayerMergedGameWeek[round]),
        AllPlayerMergedGameWeek[season] = curseason)
VAR targetgw = gameweek - 5

VAR playersThisSeason = 
    FILTER(
        VALUES(AllPlayerMergedGameWeek[Playerid.PlayerID]),
        CALCULATE(
            COUNTROWS(AllPlayerMergedGameWeek),
            AllPlayerMergedGameWeek[season] = curseason,
            AllPlayerMergedGameWeek[teamid] = currentteam,
            AllPlayerMergedGameWeek[round] >= targetgw,
            AllPlayerMergedGameWeek[minutes] > 0
        ) > 0)

VAR playertable = 
    SUMMARIZE(
        ALL(AllPlayerMergedGameWeek),
        AllPlayerMergedGameWeek[Playerid.PlayerID],
        AllPlayerMergedGameWeek[name],
        AllPlayerMergedGameWeek[teamid],
        AllPlayerMergedGameWeek[oppid],
        "totalpoints",
        SUM(AllPlayerMergedGameWeek[total_points]))

VAR filteredBySeasonPlayers = 
    FILTER(
        playertable,
        [Playerid.PlayerID] IN playersThisSeason &&
        [teamid] = currentteam
    )

VAR topscorer = 
    TOPN(
        1,
        FILTER(filteredBySeasonPlayers,
            [teamid] = currentteam &&
            [oppid] = currentopp
        ),
        [totalpoints],
        DESC)

VAR playername = MAXX(topscorer, [Playerid.PlayerID])
VAR namefinal = CALCULATE(SELECTEDVALUE(Playerid[web_name]), Playerid[PlayerID] = playername)
VAR points = MAXX(topscorer, [totalpoints])

RETURN 
    IF(ISBLANK(points), "no history", namefinal & ": " & points & " points")
```
## Captain Score

### Purpose

A composite score that ranks players by their likelihood to return high points. It combines form, underlying metrics, fixture difficulty, and playing time into a single number. Higher scores indicate better captain picks.

### How It Works

**Step 1: Define Component Weights**

The measure combines five factors with different importance levels:
- **Form (weight 3)**: Recent points scored—important but not dominant
- **Underlying Metrics (weight 4.5)**: xGI + ICT Index—highest weight because it's most predictive
- **Fixture Difficulty (weight 2)**: Opponent strength—easier fixtures = better opportunity
- **Playing Time (weight 4)**: Minutes played—must be in the team to score
- **Team Strength (weight 1, negative)**: Opponent's attacking strength—subtracted because strong teams are harder to score against

**Step 2: Get Raw Values**

Retrieves the five component scores you've already calculated: Form Score, Underlying Metrics, Next Fixture Difficulty, Player Minutes Score, and Team Difficulty.

**Step 3: Normalize Each Component**

Each component uses a different scale (form 0-30, xGI 0-5, difficulty 0-15, etc.), so they're normalized to a comparable range:
- **Form**: Divided by 20 and raised to power 1.2 to slightly boost high-form players
- **ICT, Difficulty, Minutes, Team Strength**: Divided by their typical max value (10, 5, 5, 5) to normalize to 0-1 range
- **Blank Handling**: If any metric is missing, defaults to 1 (neutral)

**Step 4: Apply Weights and Combine**

Multiplies each normalized score by its weight, then adds them together. Subtracts team strength (because strong opponent = harder to score).

**Step 5: Scale Final Score**

Divides by 10 to produce a final score typically in the 0-10 range. If the score is negative (shouldn't happen), returns 0.

### DAX Code
```dax
Captain Score = 
VAR FormWeight = 3
VAR ICTWeight = 4.5
VAR DifficultyWeight = 2
VAR Minsweight = 4
VAR teamstrengthweight = 1

VAR FormRaw = [Player Form Score]
VAR ICTRaw = [Player underlyingmetrics]
VAR DiffRaw = [11nextfixdif]
VAR MinRaw = [Player Minutes Score]
VAR TeamStrengthRaw = [TeamDifficulty]

VAR FormNorm = 
    VAR BaseForm = FormRaw / 20
    RETURN 
        IF(
            ISBLANK(BaseForm) || BaseForm <= 0,
            0,
            POWER(BaseForm, 1.2)
        )

VAR ICTNorm = IF(ISBLANK(ICTRaw), 1, ICTRaw / 10)
VAR DiffNorm = IF(ISBLANK(DiffRaw), 1, DiffRaw / 5)
VAR MinNorm = IF(ISBLANK(MinRaw), 1, MinRaw / 5)
VAR TeamNorm = IF(ISBLANK(TeamStrengthRaw), 1, TeamStrengthRaw / 5)

VAR WeightedScore = 
    (((FormNorm * FormWeight) +
    (ICTNorm * ICTWeight)) +
    (DiffNorm * DifficultyWeight) -
    (TeamNorm * TeamStrengthWeight) +
    (MinNorm * Minsweight))

RETURN 
    IF(
        ISBLANK(WeightedScore) || WeightedScore < 0,
        0,
        WeightedScore / 10
    )
```

### Why This Weighting Strategy Works

**Underlying Metrics Get Highest Weight (4.5):**

xGI and ICT are most predictive of future points because they measure quality of chance creation, not luck. A player with high xGI but low points will likely score next week. This is backed by FPL research—underlying metrics outperform recent form alone.

**Form Gets Secondary Weight (3):**

Form matters, but it's noisier than underlying metrics. A player scoring 10 points in one week might not repeat, but if they're also creating high-value chances, it's more sustainable.

**Playing Time Weighted Heavy (4):**

If a player isn't playing, they can't score. Minutes played is a constraint that overrides other factors.

**Fixture Difficulty Positive, Opponent Strength Negative:**

Easy opponent = higher score. Strong opponent = lower score. This factor acts as a multiplier on base ability.

**POWER Function for Form:**

The `POWER(BaseForm, 1.2)` slightly exaggerates differences in form. A player with form 0.8 becomes 0.83 (slightly higher). This highlights the gap between in-form and out-of-form players.

### Use Cases

- **Gameweek Captain Selection**: Pick the highest-scoring player for captain armband
- **Player Comparisons**: Compare two players' captain potential across gameweeks
- **Trend Analysis**: See if a player's captaincy appeal is improving or declining

## Captain Score

### Purpose

A composite score that ranks players by their likelihood to return high points. It combines form, underlying metrics, fixture difficulty, and playing time into a single number. Higher scores indicate better captain picks.

### How It Works

**Step 1: Define Component Weights**

The measure combines five factors with different importance levels:
- **Form (weight 3)**: Recent points scored—important but not dominant
- **Underlying Metrics (weight 4.5)**: xGI + ICT Index—highest weight because it's most predictive
- **Fixture Difficulty (weight 2)**: Opponent strength—easier fixtures = better opportunity
- **Playing Time (weight 4)**: Minutes played—must be in the team to score
- **Team Strength (weight 1, negative)**: Opponent's attacking strength—subtracted because strong teams are harder to score against

**Step 2: Get Raw Values**

Retrieves the five component scores you've already calculated: Form Score, Underlying Metrics, Next Fixture Difficulty, Player Minutes Score, and Team Difficulty.

**Step 3: Normalize Each Component**

Each component uses a different scale (form 0-30, xGI 0-5, difficulty 0-15, etc.), so they're normalized to a comparable range:
- **Form**: Divided by 20 and raised to power 1.2 to slightly boost high-form players
- **ICT, Difficulty, Minutes, Team Strength**: Divided by their typical max value (10, 5, 5, 5) to normalize to 0-1 range
- **Blank Handling**: If any metric is missing, defaults to 1 (neutral)

**Step 4: Apply Weights and Combine**

Multiplies each normalized score by its weight, then adds them together. Subtracts team strength (because strong opponent = harder to score).

**Step 5: Scale Final Score**

Divides by 10 to produce a final score typically in the 0-10 range. If the score is negative (shouldn't happen), returns 0.

### DAX Code
```dax
Captain Score = 
VAR FormWeight = 3
VAR ICTWeight = 4.5
VAR DifficultyWeight = 2
VAR Minsweight = 4
VAR teamstrengthweight = 1

VAR FormRaw = [Player Form Score]
VAR ICTRaw = [Player underlyingmetrics]
VAR DiffRaw = [11nextfixdif]
VAR MinRaw = [Player Minutes Score]
VAR TeamStrengthRaw = [TeamDifficulty]

VAR FormNorm = 
    VAR BaseForm = FormRaw / 20
    RETURN 
        IF(
            ISBLANK(BaseForm) || BaseForm <= 0,
            0,
            POWER(BaseForm, 1.2)
        )

VAR ICTNorm = IF(ISBLANK(ICTRaw), 1, ICTRaw / 10)
VAR DiffNorm = IF(ISBLANK(DiffRaw), 1, DiffRaw / 5)
VAR MinNorm = IF(ISBLANK(MinRaw), 1, MinRaw / 5)
VAR TeamNorm = IF(ISBLANK(TeamStrengthRaw), 1, TeamStrengthRaw / 5)

VAR WeightedScore = 
    (((FormNorm * FormWeight) +
    (ICTNorm * ICTWeight)) +
    (DiffNorm * DifficultyWeight) -
    (TeamNorm * TeamStrengthWeight) +
    (MinNorm * Minsweight))

RETURN 
    IF(
        ISBLANK(WeightedScore) || WeightedScore < 0,
        0,
        WeightedScore / 10
    )
```

### Why This Weighting Strategy Works

**Underlying Metrics Get Highest Weight (4.5):**

xGI and ICT are most predictive of future points because they measure quality of chance creation, not luck. A player with high xGI but low points will likely score next week. This is backed by FPL research—underlying metrics outperform recent form alone.

**Form Gets Secondary Weight (3):**

Form matters, but it's noisier than underlying metrics. A player scoring 10 points in one week might not repeat, but if they're also creating high-value chances, it's more sustainable.

**Playing Time Weighted Heavy (4):**

If a player isn't playing, they can't score. Minutes played is a constraint that overrides other factors.

**Fixture Difficulty Positive, Opponent Strength Negative:**

Easy opponent = higher score. Strong opponent = lower score. This factor acts as a multiplier on base ability.

**POWER Function for Form:**

The `POWER(BaseForm, 1.2)` slightly exaggerates differences in form. A player with form 0.8 becomes 0.83 (slightly higher). This highlights the gap between in-form and out-of-form players.

### Use Cases

- **Gameweek Captain Selection**: Pick the highest-scoring player for captain armband
- **Player Comparisons**: Compare two players' captain potential across gameweeks
- **Trend Analysis**: See if a player's captaincy appeal is improving or declining


## Easiest Fixtures (HTML Card Visual)

### Purpose

This measure identifies which team has the easiest upcoming fixture list and renders it as an interactive HTML card. It calculates the cumulative fixture difficulty across all future gameweeks for each team, ranks them, and displays the team with the lowest total difficulty alongside their crest in a styled card format.

### The Logic

**Step 1: Identify Gameweek Range**

The measure first determines the min and max gameweeks in the future fixtures table, establishing the window for difficulty calculation.

Same Logic with easiest and hardest next 5 however the use maxgw = mingw +4 
**Step 2: Calculate Total Difficulty Per Team**

For each team, it sums the opponent difficulty score across all gameweeks in the range. This uses `GENERATESERIES` to iterate through every gameweek and calls your `Opponent Difficulty Matrix` measure for each team-gameweek combination. This cumulative approach surfaces which team has the easiest overall stretch.

**Step 3: Rank and Extract**

Uses `TOPN` to rank teams by cumulative difficulty in descending order, then extracts the team with the easiest fixtures (lowest total difficulty). The measure also retrieves the team name and logo URL for rendering.

**Step 4: Render HTML Card**

Generates a styled HTML card displaying the team name, FDR score, and team crest. The card uses CSS gradients, flexbox layout, and layered design for a modern, polished appearance.

### Key Technical Decisions

**Using GENERATESERIES for Iteration:**

Rather than relying on a calendar table, the measure uses `GENERATESERIES(minGW, maxGW, 1)` to create a dynamic row for each gameweek. This is efficient and doesn't require external dependencies. Each iteration calls your `Opponent Difficulty Matrix` measure, which recalculates context for that specific team and gameweek.


### DAX Code
```dax
htmleasiest range1 = 
VAR minGW = MIN(futurfixures[gw])
VAR maxGW = MAX(futurfixures[gw])
VAR topteam = 
    TOPN(
        1,
        ADDCOLUMNS(
            VALUES(futurfixures[teamid]),
            "@oppdiffscore", 
                VAR CurrentTeamID = futurfixures[teamid]
                RETURN
                    SUMX(
                        FILTER(
                            GENERATESERIES(minGW, maxGW, 1),
                            TRUE()
                        ),
                        VAR CurrentGW = [Value]
                        RETURN
                            CALCULATE(
                                [2Opponent Difficulty Matrix],
                                FILTER(
                                    ALL(allteams[code]),
                                    allteams[code] = CurrentTeamID
                                ),
                                futurfixures[gw] = CurrentGW
                            )
                    )
        ),
        [@oppdiffscore],
        DESC
    )
VAR teamid = MINX(topteam, futurfixures[teamid])
VAR diffscore = ROUND(MINX(topteam, [@oppdiffscore]), 1)
VAR teamname = 
    CALCULATE(
        SELECTEDVALUE(allteams[name]),
        FILTER(
            ALL(allteams[code]),
            allteams[code] = teamid
        )
    )
VAR TeamLogo = 
    CALCULATE(
        SELECTEDVALUE(allteams[logoteam]),
        FILTER(
            ALL(allteams[code]),
            allteams[code] = teamid
        )
    )
RETURN
"<div style='font-family: ""DIN""; background: linear-gradient(135deg, #8B7AC7 0%, #9B98D0 100%); border-radius: 20px; padding: 20px; border: 1px solid #8B7AC7; position: relative; overflow: hidden; height: 230px; display: flex; align-items: center;'>
    <div style='flex: 1; display: flex; flex-direction: column; justify-content: center; gap: 6px; padding-right: 15px; z-index: 2;'>
        <div style='margin-bottom: 8px;'><p style='margin: 0; font-size: 26px; font-weight: 600; color: rgba(255, 255, 255, 0.85); letter-spacing: 0.5px; text-transform: uppercase;'>Easiest Fixtures</p></div>
        <h1 style='margin: 0; font-size: 52px; font-weight: 800; color: #FFFFFF; line-height: 1; letter-spacing: -0.5px;'>" & teamname & "</h1>
        <div style='display: flex; align-items: baseline; gap: 8px; margin-top: 2px;'><span style='font-size: 68px; font-weight: 800; color: #4ADE80; line-height: 1; letter-spacing: -1px;'>" & FORMAT(diffscore, "0.0") & "</span><span style='font-size: 28px; font-weight: 600; color: rgba(255, 255, 255, 0.9); text-transform: uppercase; letter-spacing: 0.3px;'>FDR Score</span></div>
    </div>
    <div style='position: absolute; right: 15px; bottom: 15px; z-index: 1;'><div style='width: 160px; height: 160px; border-radius: 50%; background: radial-gradient(circle at 30% 30%, #FFFFFF 0%, #F5F5F5 50%, #E8E8E8 100%); display: flex; align-items: center; justify-content: center; box-shadow: 0 10px 32px rgba(0, 0, 0, 0.2);'><img src='" & TeamLogo & "' style='width: 130px; height: 130px; object-fit: contain; display: block;' alt='Team'/></div></div>
    <div style='position: absolute; top: -30px; right: -30px; width: 150px; height: 150px; border-radius: 50%; background: radial-gradient(circle, rgba(255, 255, 255, 0.08) 0%, rgba(255, 255, 255, 0) 70%); pointer-events: none;'></div>
</div>"
```

## Next Fixture Difficulty

Same logic as matrix -  will turn the body of this into one measure rather than repeating the calculation in different measures

11nextfixdif = 
VAR playerid = SELECTEDVALUE(Playerid[PlayerID])
VAR season = SELECTEDVALUE(Seasons[Season])
VAR gameweek = MAX(AllPlayerMergedGameWeek[round])
VAR latestgw = CALCULATE(MAX(AllPlayerMergedGameWeek[round]), AllPlayerMergedGameWeek[season] = season, ALL(AllPlayerMergedGameWeek))
VAR playerteamid = CALCULATE(MAX(AllPlayerMergedGameWeek[teamid]), 
    AllPlayerMergedGameWeek[Playerid.PlayerID] = playerid,
    AllPlayerMergedGameWeek[round] = gameweek,
    AllPlayerMergedGameWeek[season] = season)
VAR nextgw = gameweek + 1
VAR ishistorical = gameweek < latestgw

VAR OpponentID =
    IF(ishistorical,
        CALCULATE(
            MAX(AllPlayerMergedGameWeek[oppid]),
            AllPlayerMergedGameWeek[round] = nextgw,
            AllPlayerMergedGameWeek[season] = season,
            AllPlayerMergedGameWeek[teamid] = playerteamid),
        CALCULATE(
            MAX(futurfixures[oppid]),
            futurfixures[teamid] = playerteamid,
            futurfixures[gw] = nextgw))

VAR OppGS = 
    CALCULATE(
        SUM(teamgw[goals scored]),
        teamgw[teamid] = OpponentID,
        teamgw[season] = season,
        teamgw[round] <= gameweek,
        ALL(AllPlayerMergedGameWeek))

VAR OppGC = 
    CALCULATE(
        SUM(teamgw[goalsconceded]),
        teamgw[teamid] = OpponentID,
        teamgw[season] = season,
        teamgw[round] <= gameweek,
        ALL(AllPlayerMergedGameWeek))

VAR Oppgd = OppGC - OppGS
VAR OppScore = SQRT(OppGS) * Oppgd

VAR isPlayerHome = IF(ishistorical,
    CALCULATE(COUNTROWS(AllPlayerMergedGameWeek), 
        AllPlayerMergedGameWeek[teamid] = playerteamid, 
        AllPlayerMergedGameWeek[round] = nextgw, 
        AllPlayerMergedGameWeek[season] = season, 
        AllPlayerMergedGameWeek[home/ away] = "home",
        ALL(AllPlayerMergedGameWeek)) > 0,
    CALCULATE(COUNTROWS(futurfixures), 
        futurfixures[teamid] = playerteamid, 
        futurfixures[gw] = nextgw, 
        futurfixures[home/away] = "True",
        ALL(futurfixures)) > 0)

VAR HomeBonus = IF(isPlayerHome, 1, 0)
VAR FinalScore = OppScore + HomeBonus

RETURN 
    (ROUND(FinalScore,2) +100)/20

## Opponent Difficulty Rating

### Purpose

This measure calculates fixture difficulty for both historical and future matches, enabling dynamic opponent strength assessment. It powers the fixture difficulty matrix visualization and drives captain recommendation logic by quantifying how challenging each upcoming opponent is.

### The Algorithm

The difficulty score is calculated using defensive and offensive statistics:

**Core Logic:**
- Start with `Goals Conceded - Goals Scored` (goal differential)
- Multiply by `√(Goals Scored)` to factor in attacking capability
- Add home advantage bonus (+1 if playing at home, +0 if away)
- Standardize by adding 100 and dividing by 20, keeping all values positive and in a reasonable range

**Why this approach:**

- **Goal Differential**: Weaker teams (more goals conceded than scored) receive higher difficulty scores, reflecting their defensive weakness as an opportunity
- **Square Root of Goals Scored**: Captures attacking quality without over-weighting high-scoring teams. Square root handles variability—a team scoring 4 vs 9 goals has meaningful difference, but isn't 2.25x as important
- **Home Bonus**: Adds +1 for home fixtures, +0 for away (home teams are generally stronger opponents)
- **Standardization**: The final transformation keeps scores positive and in a 5-15 range, making them interpretable and comparable

### DAX Code
```dax
Opponent Difficulty Matrix = 

VAR PlayerTeamID = SELECTEDVALUE(futurfixures[teamid])
VAR CurrentSeason = "2025-26"
VAR latestGW = 
    CALCULATE(
        MAX(AllPlayerMergedGameWeek[round]),
        AllPlayerMergedGameWeek[season] = CurrentSeason,
        ALL(AllPlayerMergedGameWeek)
    )

VAR targetGW = SELECTEDVALUE(futurfixures[gw])
VAR isTotal = ISBLANK(targetGW)

RETURN
IF(
    isTotal,
    // Total context - sum all fixtures for this team
    SUMX(
        FILTER(
            futurfixures,
            futurfixures[teamid] = PlayerTeamID
        ),
        VAR OpponentID = futurfixures[oppid]
        VAR isPlayerHome = futurfixures[home/away] = "True"
        VAR HomeBonus = IF(isPlayerHome, 1, 0)
        
        VAR OppGS = 
            CALCULATE(
                SUM(teamgw[goals scored]),
                teamgw[teamid] = OpponentID,
                teamgw[season] = CurrentSeason,
                ALL(allteams))
        
        VAR OppGC = 
            CALCULATE(
                SUM(teamgw[goalsconceded]),
                teamgw[teamid] = OpponentID,
                teamgw[season] = CurrentSeason,
                ALL(allteams))
        
        VAR Oppgd = OppGC - OppGS
        VAR gdfinal = IF(Oppgd = 0, 1, Oppgd)
        VAR OppScore = SQRT(OppGS) * gdfinal
        VAR FinalScore = OppScore + HomeBonus
        
        RETURN 
            (ROUND(FinalScore, 1) + 100) / 20
    ),
    
    // Single GW context
    VAR isHistorical = targetGW <= latestGW
    VAR OpponentID =
        IF(
            isHistorical,
            CALCULATE(
                MAX(teamgw[oppid]),
                teamgw[teamid] = PlayerTeamID,
                teamgw[round] = targetGW,
                teamgw[season] = CurrentSeason,
                ALL(teamgw)
            ),
            CALCULATE(
                MAX(futurfixures[oppid]),
                futurfixures[teamid] = PlayerTeamID,
                futurfixures[gw] = targetGW,
                ALL(futurfixures)
            )
        )
    VAR isPlayerHome =
        IF(
            isHistorical,
            CALCULATE(
                COUNTROWS(teamgw),
                teamgw[teamid] = PlayerTeamID,
                teamgw[round] = targetGW,
                teamgw[season] = CurrentSeason,
                teamgw[home/away] = "home",
                ALL(teamgw)
            ) > 0,
            CALCULATE(
                COUNTROWS(futurfixures),
                futurfixures[teamid] = PlayerTeamID,
                futurfixures[gw] = targetGW,
                futurfixures[home/away] = "True",
                ALL(futurfixures)
            ) > 0
        )
    VAR OppGS = 
        CALCULATE(
            SUM(teamgw[goals scored]),
            teamgw[teamid] = OpponentID,
            teamgw[season] = CurrentSeason,
            ALL(allteams))
    VAR OppGC = 
        CALCULATE(
            SUM(teamgw[goalsconceded]),
            teamgw[teamid] = OpponentID,
            teamgw[season] = CurrentSeason,
            ALL(allteams))
    VAR Oppgd = OppGC - OppGS
    VAR gdfinal = IF(Oppgd = 0, 1, Oppgd)
    VAR OppScore = SQRT(OppGS) * gdfinal
    VAR HomeBonus = IF(isPlayerHome, 1, 0)
    VAR FinalScore = OppScore + HomeBonus

    RETURN 
        (ROUND(FinalScore, 1) + 100) / 20

## Past 4 Results (Head-to-Head History)

### Purpose

This measure shows the last 4 head-to-head results between two teams. It displays results as color-coded dots (🟢 Win, 🟡 Draw, 🔴 Loss) so you can quickly see how a team has performed against their upcoming opponent historically.

### How It Works

**Step 1: Get the Teams and Current Gameweek**

The measure identifies which team you're looking at, which team they're playing against, and what gameweek we're currently in. This context comes from the filters applied to your dashboard.

**Step 2: Find Past Matches**

It searches through all historical gameweek data to find matches between these two teams. It only looks at matches that already happened from previous seasons or earlier gameweeks in the current season. 

**Step 3: Get the Last 4 Results**

Using `TOPN(4, ...)`, it retrieves the 4 most recent head-to-head matches between the teams, ordered with the most recent first.

**Step 4: Convert to Emojis**

Each result is converted to an emoji using a `SWITCH` statement:
- **🟢 Green** = Win
- **🟡 Yellow** = Draw
- **🔴 Red** = Loss

**Step 5: Display**

Results are concatenated into a string (e.g., "🟢 🟢 🟡 🔴") so you can see the pattern at a glance.

### DAX Code
```dax
past3results = 
VAR CurrentTeam = SELECTEDVALUE(futurfixtures[teamid])
VAR CurrentOpponent = SELECTEDVALUE(futurfixtures[oppid])
VAR CurrentGW = SELECTEDVALUE(futurfixtures[gw])
VAR CurrentSeasonText = SELECTEDVALUE(Seasons[Season])
VAR CurrentSeason = VALUE(LEFT(CurrentSeasonText, 4))

VAR Last3 = 
    TOPN(
        4,
        FILTER(
            ALL(teamgw),
            teamgw[teamid] = CurrentTeam &&
            teamgw[oppid] = CurrentOpponent &&
            (
                VALUE(LEFT(teamgw[season], 4)) < CurrentSeason ||
                (VALUE(LEFT(teamgw[season], 4)) = CurrentSeason && teamgw[round] < CurrentGW)
            )
        ),
        VALUE(LEFT(teamgw[season], 4)) * 1000 + teamgw[round],
        DESC
    )
RETURN 
    IF(
        ISBLANK(CurrentTeam) || ISBLANK(CurrentOpponent),
        "no history",
        IF(
            COUNTROWS(Last3) = 0,
            "no history",
            CONCATENATEX(
                Last3, 
                SWITCH(
                    teamgw[result],
                    "Win", "🟢",
                    "Draw", "🟡",
                    "Loss", "🔴",
                    "-"
                ),
                " ",
                VALUE(LEFT(teamgw[season], 4)) * 1000 + teamgw[date],
                ASC
            )
        )
    )

## Player Form Score

### Purpose

This measure calculates a player's recent form by averaging their points over the last 3 gameweeks (weighted heavily) and the entire season so far (weighted lightly). It gives you a quick indicator of whether a player is in good form or declining, helping inform captain and transfer decisions.

### How It Works

**Step 1: Get Player and Season Context**

The measure identifies which player you're looking at and which season you're in. It also calculates the latest gameweek for that season.

**Step 2: Determine Which Gameweek to Use**

It checks if the user has filtered the dashboard to a specific gameweek. If they have, use that gameweek. If not, use the latest gameweek. This allows the measure to work whether someone is viewing current form or historical form at a specific point in time.

**Step 3: Calculate Last 3 Gameweeks Average**

Calculates the average points the player scored in the last 3 gameweeks (including the current one). This is weighted at 1.5x because recent form is most important.

**Step 4: Calculate Season Average**

Calculates the average points across the entire season up to the current gameweek. This is weighted at 2.5x in order to prioritse consistant performers over the season

**Step 5: Combine and Average**

Adds the two weighted averages together and divides by 2 to get a final form score. Higher scores mean the player is in better form.

**Step 6: Handle Missing Data**

If there's no data (player hasn't played yet), returns 0.

### DAX Code
```dax
Player Form Score = 
VAR player = SELECTEDVALUE(Playerid[PlayerID])
VAR CurrentSeason = SELECTEDVALUE(Seasons[Season])
VAR latestGW = CALCULATE(
    MAX(AllPlayerMergedGameWeek[round]), 
    AllPlayerMergedGameWeek[season] = CurrentSeason
)

VAR isGWFiltered = ISFILTERED(AllPlayerMergedGameWeek[round])

VAR currentGW = IF(
    isGWFiltered,
    MAX(AllPlayerMergedGameWeek[round]),
    latestGW
)

VAR last3 = 
    CALCULATE(
        AVERAGE(AllPlayerMergedGameWeek[total_points]),
        AllPlayerMergedGameWeek[Playerid.PlayerID] = player,
        AllPlayerMergedGameWeek[round] >= currentgw - 2,
        AllPlayerMergedGameWeek[round] <= currentgw,
        AllPlayerMergedGameWeek[season] = CurrentSeason
    ) * 1

VAR last5 = 
    CALCULATE(
        AVERAGE(AllPlayerMergedGameWeek[total_points]),
        AllPlayerMergedGameWeek[Playerid.PlayerID] = player,
        AllPlayerMergedGameWeek[round] >= 1,
        AllPlayerMergedGameWeek[round] < currentgw,
        AllPlayerMergedGameWeek[season] = CurrentSeason
    ) * 4

VAR form = (last3 + last5) / 2

RETURN 
    IF(ISBLANK(form), 0, form)

## Top Scorer Card

Html content visual which calculates the top scorer in current filter context and returns Name Image and goals scored, same format for the rest of the card on overview page

}topscorer card = 
VAR TopPlayer =
    TOPN (
        1,
        SUMMARIZE (
            FILTER(AllPlayerMergedGameWeek, NOT ISBLANK(RELATED(Playerid[PlayerID]))),
            Playerid[PLAYERID],
            Playerid[web_name],
            Playerid[image_url],
            "Goals", SUM(AllPlayerMergedGameWeek[goals_scored])
        ),
        [Goals], DESC
    )
VAR TopPlayerID = MAXX(TopPlayer, Playerid[PLAYERID])
VAR PlayerName = 
    CALCULATE(
        SELECTEDVALUE(Playerid[web_name]),
        Playerid[PLAYERID] = TopPlayerID
    )
VAR ImageURL = 
    CALCULATE(
        SELECTEDVALUE(Playerid[image_url]), 
        Playerid[PLAYERID] = TopPlayerID
    )
VAR Goals = MAXX(TopPlayer, [Goals])
RETURN
"<div style='font-family: ""DIN""; background: linear-gradient(135deg, #8B7AC7 0%, #9B98D0 100%); border-radius: 24px; padding: 24px; box-shadow: 0 8px 32px rgba(0, 0, 0, 0.12); border: 1px solid #8B7AC7; position: relative; overflow: hidden; height: 280px; width: 570px; display: flex; align-items: center;'>
    
    <!-- Left Content -->
    <div style='flex: 1; display: flex; flex-direction: column; justify-content: center; gap: 8px; padding-right: 20px; z-index: 2;'>
        <!-- Title -->
        <div style='margin-bottom: 12px;'>
            <p style='margin: 0; font-size: 26px; font-weight: 600; color: rgba(255, 255, 255, 0.85); letter-spacing: 0.5px; text-transform: uppercase;'>Top Scorer</p>
        </div>
        
        <!-- Player Name -->
        <h1 style='margin: 0; font-size: 52px; font-weight: 800; color: #FFFFFF; line-height: 1; letter-spacing: -0.5px;'>" & PlayerName & "</h1>
        
        <!-- Stats -->
        <div style='display: flex; align-items: baseline; gap: 10px; margin-top: 4px;'>
            <span style='font-size: 68px; font-weight: 800; color: #FFD700; line-height: 1; letter-spacing: -1px;'>" & Goals & "</span>
            <span style='font-size: 28px; font-weight: 600; color: rgba(255, 255, 255, 0.9); text-transform: uppercase; letter-spacing: 0.5px;'>Goals</span>
        </div>
    </div>
    
    <!-- Right: Player Image (Bottom Aligned) -->
    <div style='position: absolute; right: 20px; bottom: -15px; z-index: 1;'>
        <!-- Gradient Circle Background -->
        <div style='width: 180px; height: 180px; border-radius: 50%; background: radial-gradient(circle at 30% 30%, #FFFFFF 0%, #F5F5F5 50%, #E8E8E8 100%); display: flex; align-items: center; justify-content: center; box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);'>
            <img src='" & ImageURL & "' style='width: 210px; height: 210px; object-fit: contain; display: block;' alt='Player'/>
        </div>
    </div>
    
    <!-- Decorative Element -->
    <div style='position: absolute; top: -40px; right: -40px; width: 200px; height: 200px; border-radius: 50%; background: radial-gradient(circle, rgba(255, 255, 255, 0.1) 0%, rgba(255, 255, 255, 0) 70%); pointer-events: none;'></div>
</div>"

Player Minutes Score = 
VAR playerid = SELECTEDVALUE(Playerid[PlayerID])


VAR CurrentSeason = SELECTEDVALUE(Seasons[Season])

VAR latestGW = CALCULATE(
    MAX(AllPlayerMergedGameWeek[round]), 
    AllPlayerMergedGameWeek[season] = currentseason
)

// Check if user has applied a gameweek filter
VAR isGWFiltered = ISFILTERED(AllPlayerMergedGameWeek[round])

// If filtered: use the filtered GW, otherwise use latest
VAR maxgw = IF(
    isGWFiltered,
    MAX(AllPlayerMergedGameWeek[round]),  // User filtered to a specific GW
    latestGW  // No filter, use latest GW
)
VAR totalmins = maxgw *90

// Get average minutes in last 5 gameweeks
VAR Minutes = 
    CALCULATE(
        SUM(AllPlayerMergedGameWeek[minutes]),
        AllPlayerMergedGameWeek[Playerid.PlayerID] = playerid,
        FILTER(
            ALL(AllPlayerMergedGameWeek[round]),
            AllPlayerMergedGameWeek[round] <= maxgw &&
            AllPlayerMergedGameWeek[round] > 0 
        )
    )

// Convert to score based on reliability
VAR minutesScore = 
    SWITCH(
        TRUE(),
        Minutes >= totalmins*0.85, 10,   // Nailed starter (plays full games)
        Minutes >= totalmins*0.75, 8,    // Mostly plays full games
        Minutes >= totalmins*0.65, 6,    // Regular starter
        Minutes >= totalmins*0.55, 4,    // Some rotation
        Minutes >= totalmins*0.45, 2,    // Heavy rotation
        0                       // Barely plays
    )

RETURN minutesScore


## Player Underlying Metrics

### Purpose

This measure evaluates a player's underlying performance quality using expected goals (xG) and expected assists (xA), combined with the ICT Index (Influence, Creativity, Threat). Rather than just looking at actual points scored, this shows whether a player is performing well based on chances created and involvement in play—useful for identifying players due for a breakout and spotting unsustainable form.

### How It Works

**Step 1: Get Player and Season Context**

Identifies which player you're viewing and which season you're in. Like the Form Score measure, it determines the current gameweek based on whether the user has filtered to a specific gameweek.

**Step 2: Calculate Expected Goal Involvements (xGI)**

Calculates two versions of xGI (expected goals + expected assists):
- **Last 3 gameweeks** (weighted 2x): How many goals/assists the player is expected to score/create recently
- **Full season** (weighted 6x): Their overall xGI average for the entire season

The heavier weight on season xGI (6x vs 2x) reflects the idea that underlying metrics are most stable over time—short-term variance is less meaningful.

**Step 3: Calculate ICT Index**

Similarly calculates ICT Index in two timeframes:
- **Last 3 gameweeks** (weighted 1x): Recent involvement and impact
- **Full season** (weighted 2x): Sustained level of involvement

The ICT Index measures Influence (defensive actions), Creativity (chances created), and Threat (attacking potential). It's an FPL-specific metric that captures overall player involvement.

**Step 4: Combine Scores**

Averages the weighted xGI and weighted ICT scores to create a single underlying metrics score. Higher values indicate a player is performing well on the underlying metrics.

**Step 5: Handle Missing Data**

If there's no data, returns 1 (neutral baseline) instead of blank.

### DAX Code
```dax
Player underlyingmetrics = 
VAR player = SELECTEDVALUE(Playerid[PlayerID])
VAR CurrentSeason = SELECTEDVALUE(Seasons[Season])
VAR latestGW = CALCULATE(
    MAX(AllPlayerMergedGameWeek[round]), 
    AllPlayerMergedGameWeek[season] = CurrentSeason
)

VAR isGWFiltered = ISFILTERED(AllPlayerMergedGameWeek[round])

VAR currentGW = IF(
    isGWFiltered,
    MAX(AllPlayerMergedGameWeek[round]),
    latestGW
)

VAR xGI3 = 
    CALCULATE(
        AVERAGE(AllPlayerMergedGameWeek[expected_goal_involvements]),
        AllPlayerMergedGameWeek[Playerid.PlayerID] = player,
        AllPlayerMergedGameWeek[round] >= currentgw - 2,
        AllPlayerMergedGameWeek[round] <= currentgw,
        AllPlayerMergedGameWeek[season] = CurrentSeason
    ) * 2

VAR xgi5 = 
    CALCULATE(
        AVERAGE(AllPlayerMergedGameWeek[expected_goal_involvements]),
        AllPlayerMergedGameWeek[Playerid.PlayerID] = player,
        AllPlayerMergedGameWeek[round] >= 1,
        AllPlayerMergedGameWeek[round] <= currentgw,
        AllPlayerMergedGameWeek[season] = CurrentSeason
    ) * 6

VAR ICT3 = 
    CALCULATE(
        AVERAGE(AllPlayerMergedGameWeek[ict_index]),
        AllPlayerMergedGameWeek[Playerid.PlayerID] = player,
        AllPlayerMergedGameWeek[round] >= currentgw - 2,
        AllPlayerMergedGameWeek[round] <= currentgw,
        AllPlayerMergedGameWeek[season] = CurrentSeason
    ) * 1

VAR ICT5 = 
    CALCULATE(
        AVERAGE(AllPlayerMergedGameWeek[ict_index]),
        AllPlayerMergedGameWeek[Playerid.PlayerID] = player,
        AllPlayerMergedGameWeek[round] >= 1,
        AllPlayerMergedGameWeek[round] <= currentgw,
        AllPlayerMergedGameWeek[season] = CurrentSeason
    ) * 2

VAR xgi = (xGI3 + xgi5) / 2
VAR ICT = (ICT3 + ICT5) / 2
VAR combinedScore = (xgi + ICT) / 2

RETURN 
    IF(ISBLANK(combinedScore), 1, combinedScore)
```

### Key Concepts

**Why Weight Past Form More Heavily:**

xGI uses 2x for last 3 weeks but 6x for season average. I chose this because season averages are more stable and predictive, so they get higher weight. Recent form is noisy and can vary week to week.



