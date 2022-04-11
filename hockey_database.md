# Team Hockey Database Project

In this project, I was on a team with Evan Hanh, Jongbae Yoon, Noah Lyford, and Santhosh Nair. The data used can be found [on Kaggle](https://www.kaggle.com/datasets/martinellis/nhl-game-data).

## Project Scenario

Analytics and data analysis has always been a part of sports, but in recent years the use of data to improve player and team performance has become a major business. As the business has grown the demand for more technical analysis has grown as well. These sorts of data analyses are most prevalent in baseball and basketball, but other sports like soccer, football and hockey are entering the data revolution as well. Our goal in creating our database is to create something that resembles what a National Hockey League franchise might utilize to analyze their own and other team’s players.
Our database compiles information about the National Hockey League from the 2010 season through the 2019/2020 season. The information includes player and team statistics, as well as characteristic data. While the database is likely not as complex as what a real NHL franchise might use, it is a valuable tool for answering difficult questions that would be impossible without a sophisticated database management system.
In our imagined business scenario, we are operating a single NHL franchise but need to utilize a database of the entire league in order to gain information about our opponents and their players. We can identify who the most valuable players in the league are and trade for them or recognize when a player is undervalued and sign that player to a contract. Similarly, we can identify when one of our own players is underperforming and might need to be removed from the team.

## A note on Normal Form and Dependencies

We started this project by importing data in .csv format from Kaggle, but the data was organized in a way where it was not normalized. For one, we downloaded a giant csv that had to be put in 1NF. We accomplished this by splitting the columns with multiple values into multiple columns with each of the values listed separately. Now we have Normal Form 1.
Next, we looked onward to 2NF. In order to get our data to the second Normal Form, it has to be in the first Normal Form and all primary keys must consist of one column. For example, if we had a table that included teams and their cities, then the tuple (team,city) would have to be unique. To solve this, we split the tables up into multiple tables each with a primary key that makes more sense. We split the team table into team and city to solve this issue.
Lastly, we had to get our data into 3NF. Third Normal Form means that the data is already in Second Normal Form and has no functional dependencies. Functional dependencies would cause issues in our database if we update one column’s information, but forget to update the columns that are functionally dependent on this table. In order to solve this problem, we assigned IDs to players, teams, cities, and games. We then split our tables up even more so that there would be no functional dependencies.


## CREATE Statements

```MySQL
CREATE database nhl;
use nhl;

-------------------- plays --------------------------------
DROP TABLE plays;
CREATE TABLE plays (
play_id	INT,
game_id	INT,
team_id_for INT,
team_id_against INT,
event VARCHAR(45),
secondaryType varchar(30),
x INT NULL,
y INT NULL,
period SMALLINT,
periodType VARCHAR(30),
periodTime INT,
periodTimeRemaining INT,
dateTime DATETIME,
goals_away INT,
goals_home INT,
description	VARCHAR(50),
st_x INT NULL,		
st_y INT NULL,
CONSTRAINT playID PRIMARY KEY (play_id),
CONSTRAINT gameID6 FOREIGN KEY (game_id) REFERENCES game (game_id),
CONSTRAINT teamID_F FOREIGN KEY (team_id_for) REFERENCES team (team_id),
CONSTRAINT teamID_A FOREIGN KEY (team_id_against) REFERENCES team (team_id)
);

-------------------- position --------------------------------

CREATE TABLE position(
position_ID INT,
position_name VARCHAR(45),
CONSTRAINT positionID PRIMARY KEY (position_ID)
);
-------------------- State --------------------------------

CREATE TABLE state(
state_id INT,
state_name VARCHAR(45),
CONSTRAINT stateID PRIMARY KEY (state_id)
);

-------------------- game --------------------------------

CREATE TABLE game(
game_id INT,
season INT,
game_type VARCHAR(2),
date_time_GMT DATETIME,
away_team_id INT,
home_team_id INT,
away_goals INT,
home_goals INT,
outcome VARCHAR(45),
CONSTRAINT gameID PRIMARY KEY (game_id),
CONSTRAINT hometeam FOREIGN KEY (home_team_id) REFERENCES team (team_id),
CONSTRAINT awayteam FOREIGN KEY (away_team_id) REFERENCES team (team_id)
);

-------------------- team --------------------------------

CREATE TABLE team(
team_id INT,
franchiseID_ph VARCHAR(45),
team_name VARCHAR(45),
Abbreviation VARCHAR(3),
city_id INT,
CONSTRAINT teamID PRIMARY KEY (team_id)
);

-------------------- Player --------------------------------

CREATE TABLE player(
player_id INT,
firstName VARCHAR(45),
lastName VARCHAR(45),
nationality VARCHAR(45),
primary_position_ID INT,
birthDate DATE,
height_cm DOUBLE,
CONSTRAINT playerID PRIMARY KEY (player_id),
CONSTRAINT primaryPosition FOREIGN KEY (primary_position_id) REFERENCES position (position_id)
);

-------------------- Game_teams --------------------------------

CREATE TABLE game_teams(
game_id	INT,
team_id INT,
HoA VARCHAR(10),
win	BOOLEAN,
Overtime VARCHAR(10),
head_coach VARCHAR(45),
goals INT NULL,
shots INT NULL,
hits INT NULL,
pim	INT NULL,
powerPlayOpportunities INT NULL,
powerPlayGoals INT NULL,
faceOffWinPercentage DECIMAL NULL,
CONSTRAINT gameID FOREIGN KEY (game_id) REFERENCES game (game_id),
CONSTRAINT teamID FOREIGN KEY (team_id) REFERENCES team (team_id)
);

-------------------- Game_skater_stats --------------------------------

CREATE TABLE game_skater_stats(
game_id	INT,
player_id INT,
team_id	INT,
timeOnIce INT,
assists	INT,
goals INT,
shots INT,
hits INT,
power_play_goals INT,
power_play_assist INT,
penalty_minutes	INT,
face_off_wins INT,
face_off_taken INT,
take_aways INT,
give_aways INT,
short_handed_goals INT,
short_handed_assists INT,
blocked INT,
plusMinus VARCHAR(45),
even_time_on_ice INT,
short_handed_time_on_ice INT,
pwer_play_time_on_ice INT,
CONSTRAINT gameID4 FOREIGN KEY (game_id) REFERENCES game (game_id),
CONSTRAINT playerID2 FOREIGN KEY (player_id) REFERENCES player (player_id),
CONSTRAINT teamID3 FOREIGN KEY (team_id) REFERENCES team (team_id)
);

select count(*) from game_skater_stats;

-------------------- Game_shifts --------------------------------

CREATE TABLE game_shifts(
game_id INT,
player_id INT,
period	SMALLINT,
shift_start	TIME,
shift_end TIME,
CONSTRAINT gameID3 FOREIGN KEY (game_id) REFERENCES game (game_id),
CONSTRAINT playerID FOREIGN KEY (player_id) REFERENCES player (player_id)
);

-------------------- Game_scratches --------------------------------

CREATE TABLE game_scratches(
game_id INT,
team_id INT	,
player_id INT,
CONSTRAINT gameID2 FOREIGN KEY (game_id) REFERENCES game (game_id),
CONSTRAINT teamID2 FOREIGN KEY (team_id) REFERENCES team (team_id)
);
-------------------- Game_penalties --------------------------------

CREATE TABLE game_penalties(
play_id	INT,
penalty_severity VARCHAR(45),
penalty_minutes INT,
CONSTRAINT playID2 FOREIGN KEY (play_id) REFERENCES plays (play_id)
);

-------------------- Game_goals --------------------------------

CREATE TABLE game_goals(
play_id	INT,
strength VARCHAR(2),
gameWinningGoal VARCHAR(1),
emptyNet VARCHAR(1),
CONSTRAINT playID3 FOREIGN KEY (play_id) REFERENCES plays (play_id)
);

-------------------- Game_goalies --------------------------------

CREATE TABLE game_goalies(
game_id INT,
player_id INT,
team_id INT,
timeOnIce INT,
assists INT,
goals INT,
pim INT,
shots INT,
saves INT,
powerPlaySaves INT,
shortHandedSaves INT,
evenSaves INT,
shortHandedShotsAgainst INT,
evenShotsAgainst INT,
powerPlayShotsAgainst INT,
decision SMALLINT,
savePercentage DECIMAL,
powerPlaySavePercentage DECIMAL,
evenStrengthSavePercentage DECIMAL,
CONSTRAINT gameID5 FOREIGN KEY (game_id) REFERENCES game (game_id),
CONSTRAINT playerID3 FOREIGN KEY (player_id) REFERENCES player (player_id),
CONSTRAINT teamID4 FOREIGN KEY (team_id) REFERENCES team (team_id)
);

-------------------- Play_players --------------------------------

CREATE TABLE play_players(
play_id INT,
game_id	INT,
player_id INT,
playerType VARCHAR(20),
CONSTRAINT gameID_7 FOREIGN KEY (game_id) REFERENCES game (game_id),
CONSTRAINT playerID4 FOREIGN KEY (player_id) REFERENCES player (player_id)
);

-------------------- city --------------------------------
CREATE TABLE IF NOT EXISTS city(
  city_id INT NOT NULL,
  city_name VARCHAR(45) NULL,
  state_id INT NOT NULL,
  CONSTRAINT cityID PRIMARY KEY (city_id),
  CONSTRAINT stateID2 FOREIGN KEY (state_id) REFERENCES state (state_id)
  );
  ```

The city and state tables are small, containing the city and state information of teams. We used one to many relationships to connect various tables. There is a many to many relationship between team and game tables using a bridge table game_scratches. game_skater_stats and game_goalies tables mostly hold all game information. game_stater_stats has almost a million observations which give every performance data required for the analysis of any individual or team based performance.

## Entity Relationship Model

<img src="images/hockey_database/hockey_ER.jpg?raw=true"/>

## Database Analysis / SELECT Statements

Below is a **taste** of the analysis we conducted on our database. I say **taste** because including all of the `SELECT` statements would be redundant for showing off my SQL abilities.

```MySQL
DELIMITER &&
CREATE FUNCTION get_winner(
	gameid INTEGER
)
RETURNS INTEGER
DETERMINISTIC
BEGIN
	DECLARE winner INTEGER;
   SELECT (CASE
			WHEN outcome LIKE 'home win%' THEN home_team_id
            WHEN outcome LIKE 'away win%' THEN away_team_id
            ELSE NULL
		END) AS winTeam
        INTO winner
        FROM game
        WHERE game_id = gameid;
    RETURN winner;
END; &&

#Average penalty_minutes per win/loss
# winners 8.7575
SELECT AVG(penMins)
FROM (SELECT SUM(penalty_minutes) AS penMins
FROM game AS g
JOIN game_skater_stats USING(game_id)
WHERE (get_winner(game_id)) = team_id
GROUP BY game_id) AS winnerTable;


#Losers 9.099 mins
SELECT AVG(penMins)
FROM (SELECT SUM(penalty_minutes) AS penMins
FROM game AS g
JOIN game_skater_stats USING(game_id)
WHERE (get_winner(game_id)) != team_id
GROUP BY game_id) AS loserTable;
```

These two queries work in tandem with each other and the associated stored function ‘get_winner’.  The first one gets the average amount of time a player is in a penalty box per game for only the winning teams, and the other is the same for losing teams.  Winning teams spend on average 8.75 minutes and losing teams spend 9.09 minutes, meaning that the longer you have a player in the penalty box, the more likely you are to lose.

```MySQL
#Average penalty time per game per team per season

SELECT season, c.city_name, t.team_name, AVG(total_pen_mins) AS 'Average Penalty Minutes'
FROM (
	SELECT season, gss.game_id, team_id, SUM(penalty_minutes) AS total_pen_mins
  FROM game_skater_stats AS gss
	JOIN game AS g
  ON gss.game_id = g.game_id
	GROUP BY game_id, team_id
    ) AS penMinGame
JOIN team AS t ON penMinGame.team_id = t.team_id
JOIN city AS c ON c.city_id = t.city_id
GROUP BY season, t.team_id
ORDER BY t.team_id, season;

#Average shots per game for skaters for most winning team?
#Utilized a subquery to find average shots for most winning team. Very useful as a goal for other teams.
SELECT t.team_name, AVG(gt.shots)
FROM game_teams AS gt
JOIN team AS t
WHERE t.team_name = (
					SELECT t.team_name
					FROM team AS t
					JOIN game_teams AS gt
					USING(team_id)
					GROUP BY t.team_name
					ORDER BY SUM(gt.win) DESC
					LIMIT 1
                    );

#Top 10 most efficient players (i.e. goal / shots) in the year 2017

SELECT s.player_id, CONCAT(p.firstName, p.lastName) AS player_Name, (goals / shots) AS efficiency
FROM game_skater_stats as s
LEFT JOIN game as g
USING(game_id)
LEFT JOIN player as p
USING(player_id)
WHERE YEAR(date_time_GMT) = 2017
ORDER BY efficiency DESC
LIMIT 10;

#players who played more than 120,000 hours in the year 2017

SELECT distinct(s.player_id) AS id, CONCAT(p.firstName, p.lastName) AS Full_Name, SUM(timeOnIce) AS playTime
FROM game_skater_stats AS s
LEFT JOIN game AS g
USING(game_id)
LEFT JOIN player AS p
USING(player_id)
WHERE YEAR(date_time_GMT) = 2017
GROUP BY id, Full_Name
HAVING sum(timeOnIce) > 120000
ORDER BY playTime DESC;

#How many United states players have height above the average height of Canadian players.

SELECT count(player_id) FROM player
WHERE height_cm >
(SELECT AVG(height_cm) FROM player WHERE nationality='CAN')
AND nationality= 'USA';
```
## Visualizations using Tableau

<img src="images/hockey_database/most_scoring_teams.jpg?raw=true"/>

This shows the teams that score the most in our dataset using a fun word map. Unfortunately for some of the newer teams, it is hard to catch up to the mainstay teams like the Lightning, Islanders, Maple Leafs and Penguins.

<img src="images/hockey_database/shots_vs_saves.jpg?raw=true"/>

Using a bar plot to show the difference between shots made on and saves for each goalie.

<img src="images/hockey_database/total_goals_team.jpg?raw=true"/>

I wanted to show the total goals by each team because it shows which team is more aggressive and this means their games are more fun to watch. I  joined game_skater_stats table and teams table using team_di, and calculated the sum of the goals and order by it. I used a bar chart.

<img src="images/hockey_database/height_hist.jpg?raw=true"/>

This visualization shows the distribution of players’ height. It has normal distribution, and the average height is approximately 185.42 cm, which is 6’1’’.

<img src="images/hockey_database/scores_by_position.jpg?raw=true"/>

This visualization shows how goals are distributed when a team wins. Basically, it's a visualization of how likely our team is to win if we scored x amount of goals. This was created by breaking up the goals scored into bins, and then filtering for a win.

<img src="images/hockey_database/avalanche_goals_monthly.jpg?raw=true"/>

My final visualization shows the average goals per month per year for just the Avalanche. This was created by joining the game_teams table, with the game table and the team table. This gives us a sense of how our team has been performing over the last few seasons.
