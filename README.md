# MLB API Database Builder
Tools for acquiring data and building databases from MLB Stats API Data (2021 and 2022 Data).

This Python code sets the framework for reading MLB Stats API data into a Pandas dataframe where it can be manipulated and later written to a database for analysis. Includes helper methods to make calls to get API data from 5 endpoints: GAME, PEOPLE, TEAM, SCHEDULE, SPORTS. Each returned data set is then cleaned and loaded into a Pandas dataframe for further manipulation/analysis. In the current iteration, chosen values/categories are custom for a previous project, but they can be adjusted to suit any data needs. API data is processed and transformed into four SQLite3 database tables (2021 Game Events, 2022 Game Events, 2021 Players, and 2022 Players) which can be used to answer types of questions such as the following (with SQL queries attached):

### On July 8th, 2022, which pitches happened at > 95mph, who threw the pitch, how old were they, and what team did they play for?

select Game_Data_2022.gameId, count(Game_Data_2022.pitchSpeed), Game_Data_2022.pitchType, Player_Data_2022.fullName, Player_Data_2022.age, Player_Data_2022.team from Game_Data_2022 JOIN Player_Data_2022 ON Game_Data_2022.pitcherId=Player_Data_2022.id where Game_Data_2022.pitchSpeed > 95 and Game_Data_2022.gameDate='2022-07-08' group by Player_Data_2022.fullName order by count(Game_Data_2022.pitchSpeed) desc

### Given these pitchers, what percentage of pitches did they throw over 95mph for the season?

SELECT allpitchestab.fullName, over95col/allpitchescol as percentover95
FROM (SELECT over95.fullName, 1.0 * count(over95.fullName) as over95col
FROM (SELECT  Player_Data_2022.fullName from Game_Data_2022 JOIN Player_Data_2022 ON Game_Data_2022.pitcherId=Player_Data_2022.id where Player_Data_2022.fullName in 
(select DISTINCT(Player_Data_2022.fullName) 
from Game_Data_2022 JOIN Player_Data_2022 ON Game_Data_2022.pitcherId=Player_Data_2022.id 
where Game_Data_2022.pitchSpeed > 95 and Game_Data_2022.gameDate='2022-07-08') group by Player_Data_2022.fullName
) AS allpitches
LEFT JOIN (SELECT  Player_Data_2022.fullName from Game_Data_2022 JOIN Player_Data_2022 ON Game_Data_2022.pitcherId=Player_Data_2022.id
where Game_Data_2022.pitchSpeed > 95 and Player_Data_2022.fullName in 
(select DISTINCT(Player_Data_2022.fullName) 
from Game_Data_2022 JOIN Player_Data_2022 ON Game_Data_2022.pitcherId=Player_Data_2022.id 
where Game_Data_2022.pitchSpeed > 95 and Game_Data_2022.gameDate='2022-07-08') 
) AS over95
ON over95.fullName = allpitches.fullName group by allpitches.fullName) as over95tab
JOIN 
(SELECT over95.fullName, count(allpitches.fullName) as allpitchescol
FROM (SELECT  Player_Data_2022.fullName from Game_Data_2022 JOIN Player_Data_2022 ON Game_Data_2022.pitcherId=Player_Data_2022.id where Player_Data_2022.fullName in 
(select DISTINCT(Player_Data_2022.fullName) 
from Game_Data_2022 JOIN Player_Data_2022 ON Game_Data_2022.pitcherId=Player_Data_2022.id 
where Game_Data_2022.pitchSpeed > 95 and Game_Data_2022.gameDate='2022-07-08')
) AS allpitches
LEFT JOIN (SELECT  Player_Data_2022.fullName from Game_Data_2022 JOIN Player_Data_2022 ON Game_Data_2022.pitcherId=Player_Data_2022.id
where Game_Data_2022.pitchSpeed > 95 and Player_Data_2022.fullName in 
(select DISTINCT(Player_Data_2022.fullName) 
from Game_Data_2022 JOIN Player_Data_2022 ON Game_Data_2022.pitcherId=Player_Data_2022.id 
where Game_Data_2022.pitchSpeed > 95 and Game_Data_2022.gameDate='2022-07-08') group by Player_Data_2022.fullName
) AS over95
ON over95.fullName = allpitches.fullName group by allpitches.fullName) as allpitchestab
ON over95tab.fullName=allpitchestab.fullName group by allpitchestab.fullName

### Who are the top 10 hitters in average exit velocity on fastballs in the top half of the zone?

select Player_Data_2022.fullName, avg(Game_Data_2022.exitVelocity) as avgexitvelo from Game_Data_2022 JOIN
Player_Data_2022 on Game_Data_2022.batterId=Player_Data_2022.id 
where Game_Data_2022.pitchType in ('FA','FF','FT','FC') 
and Game_Data_2022.pitchLocationZ>(((Player_Data_2022.strikeZoneTop - Player_Data_2022.strikeZoneBottom) / 2) + Player_Data_2022.strikeZoneBottom) 
and Game_Data_2022.pitchLocationZ<Player_Data_2022.strikeZoneTop 
and Game_Data_2022.pitchLocationX> (-1.0 * 0.708)
and Game_Data_2022.pitchLocationX< 0.708
group by Player_Data_2022.fullName HAVING Player_Data_2022.fullName in (select Player_Data_2022.fullName from Game_Data_2022 JOIN
Player_Data_2022 on Game_Data_2022.batterId=Player_Data_2022.id 
where (Game_Data_2022.isInPlay or (Game_Data_2022.isBall and Game_Data_2022.balls=4) or (Game_Data_2022.isStrike and Game_Data_2022.strikes=3))
group by Player_Data_2022.fullName HAVING count(Player_Data_2022.fullName)>502) order by avgexitvelo DESC limit 10

**ASSUMED PITCHES OVER THE PLATE (IN THE ZONE) AND QUALIFIED HITTERS**

### Which percentile is this hitter in for average exit velocity amongst qualifying hitters under the age of 26?

select Player_Data_2022.fullName, avg(Game_Data_2022.exitVelocity) as avgexitvelo, 100 * PERCENT_RANK() OVER(ORDER BY avg(Game_Data_2022.exitVelocity)) AS percent_rank 
from Game_Data_2022 JOIN Player_Data_2022 on Game_Data_2022.batterId=Player_Data_2022.id 
where Player_Data_2022.age<26
group by Player_Data_2022.fullName HAVING Player_Data_2022.fullName in (select Player_Data_2022.fullName from Game_Data_2022 JOIN
Player_Data_2022 on Game_Data_2022.batterId=Player_Data_2022.id 
where (Game_Data_2022.isInPlay or (Game_Data_2022.isBall and Game_Data_2022.balls=4) or (Game_Data_2022.isStrike and Game_Data_2022.strikes=3))
group by Player_Data_2022.fullName HAVING count(Player_Data_2022.fullName)>502) order by avgexitvelo DESC


NOTE: Database is not included due to size restrictions. However, it can be easily regenerated from the code.
