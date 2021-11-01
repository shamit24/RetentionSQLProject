# JunoProject1 - SQL & Sheets Project: Player Retention Analysis


## Project Introduction/Problem

A mobile game company hired us to analyze their data using our SQL knowledge and expertise. The main objective is to investigate player retention and answer questions around the retention of players throughout the game life cycle. Of specific interest is counting rolling 30-day retention and expressing it as a fraction of the total playerbase at the time. 


## Exploring the Dataset

The dataset was a relational database containing 4 files.

  1. `player_info`: this table consists of player information, including information like the player's age and when they joined.

Row |	player_id	| location	| age	| system | joined	
--- | --------- | --------- | --- | ------ | ------
1	  | db8bea6bf3c94f7eae81cfd29d6a87b6 | Asia | 24 | Linux | 1

  2. `matches_info`: this table consists of information on every match played, including the players who matched against each other, and the outcome.

Row	| player_id |	match_id | opponent_id | outcome | day	
--- | --------- | -------- | ----------- | ------- | ---
1	  | b41d2a88038a491e848313ea91a0ff7f | 910afbd82c48482c9f693027f0c99b55 | 1d89dd4ffa844ff48e0b7396277ffba0 | win | 1


  3. `purchase_info`: this table consists of information on purchases from the item shop, including what player ID bought what item ID, and on what day
  
Row | player_id |	item_id |	day	
--- | --------- | ------- | ---
1	  | f8487091a7e74d19a400ccdb0459af7a | 94e22dcc4c4647409cb42de8cc30c66a | 34

  4. `item_info`: this table consists of information about the items in the store,  including the item ID and the price

Row |	item_id |	price	
--- | ------- | -----
1	  | 05ea3b238c0c4861a44d82d72c67d0d8 | 2.0


We noticed that columns day and joined were numerical with integers ranging from 1 to 365. This lead us to the assumption that each record in those columns represented a day in the lifespan of the game, which in was 365 days in the dataset. We realized we needed to use both the day and joined columns to calculate player retention. 


## Query Explanation

``` sql
SELECT 
  DISTINCT joining_day as day, 
  COUNT(player_id) OVER (PARTITION BY joining_day) players_joined, 
  SUM(retained) OVER (PARTITION BY joining_day) AS players_retained, 
  SUM(retained) OVER (PARTITION BY joining_day) / COUNT(player_id) OVER (PARTITION BY joining_day) fractional_retention 
FROM 
  (
    SELECT 
      days_info.joining_day, 
      days_info.player_id as player_id, 
      CASE WHEN days_info.lastplay_day - days_info.joining_day > 30 THEN 1 ELSE 0 END AS retained 
    FROM 
      (
        SELECT 
          players.player_id, 
          players.joined as joining_day, 
          lastplay.lastplay_day 
        FROM 
          `junoproject1.mobilegamedata.player_info` players 
          JOIN (
            SELECT 
              player_id as player_lastday, 
              MAX(day) as lastplay_day 
            FROM 
              `junoproject1.mobilegamedata.matches_info` 
            GROUP BY 
              1
          ) as lastplay ON players.player_id = lastplay.player_lastday
      ) as days_info
  ) 
ORDER BY 
  1
```

This is a nested query that outputs a single table containing the day, number of players that joined the game for a specific day, number of players retained for that day and the  retention rate. This query uses the `player_info`and `matches_info` tables.

Given that retention happens if a player plays a match 30 days after they joined the game, our first task was to figure out the last match a player played in our dataset. To do that we used the `MAX` function on our day column in the `matches_info` table.
Next we joined the matches_info and player_info tables  using an `INNER JOIN` using each tables' `player_id` as the unique key to join on
This query was then nested into another query where we established our retention logic and added an extra column that designates whether each players was retained or not. Using a `CASE` stetment we denoted a player that was retained as 1 and a player that was not retained as 0.
Finally this query is wrapped and nested into a final select statement that gives us our final output table.

The final table looks like:

Row	| day |	players_joined | players_retained |	fractional_retention
--- | --- | -------------- | ---------------- | --------------------
1	 | 1 | 113 | 75 | 0.6637168141592921


Visualizing the retention trend

![Retention per day](https://github.com/shamit24/RetentionSQLProject/blob/4b3c37be1f4d8ba5e98b67be0a07430327d62bdd/Players%20Retained%20per%20Day.png)

Link to Sheets: https://docs.google.com/spreadsheets/d/1ZI6F-oPvFeFWFFtY2iy2gbvRlU8PVGmghpxvGHDekWc/edit?usp=sharing

### Do players with rolling 30-day retention spend more?

```sql

SELECT 
  player_status, 
  SUM(amount_spent) as total_spent 
FROM 
  (
    SELECT 
      * 
    FROM 
      (
        SELECT 
          player_id as pid, 
          SUM(price) as amount_spent 
        FROM 
          `junoproject1.mobilegamedata.item_info` i 
          JOIN `junoproject1.mobilegamedata.purchase_info` p ON i.item_id = p.item_id 
        GROUP BY 
          1
      ) AS player_spent 
      JOIN (
        SELECT 
          days_info.player_id as player_id, 
          CASE WHEN days_info.lastplay_day - days_info.joining_day > 30 THEN "retained" ELSE 'unretained' END AS player_status 
        FROM 
          (
            SELECT 
              players.player_id, 
              players.joined as joining_day, 
              lastplay.lastplay_day 
            FROM 
              `junoproject1.mobilegamedata.player_info` players 
              JOIN (
                SELECT 
                  player_id as player_lastday, 
                  MAX(day) as lastplay_day 
                FROM 
                  `junoproject1.mobilegamedata.matches_info` 
                GROUP BY 
                  1
              ) as lastplay ON players.player_id = lastplay.player_lastday
          ) as days_info
      ) as player_status ON player_status.player_id = player_spent.pid
  ) as table1 
GROUP BY 
  1 
ORDER BY 
  2 DESC
```
This query answers the question of whether players with rolling 30-day retention spend more. We use tables player_info, matches_info and purchase_info to get our insights.
* First we join `matches_info` and `player_info` tables while getting the day of the last match played per `player_id` and output the columns necessary for calculating player retention.
* Then we nest this query into the outer query that adds our retention column with the `CASE` statement.
* Next we join `purchase_info` and `item_info` tables together and `SUM` the price column to find the amount spent per player by grouping the query by `player_id`
* Finally, we combine both parts of the query and aggregate over player_status to find total amount spent by grouped by retained and unretained players.

This visual will help answer the question: Do players with rolling 30-day retention spend more?

![Total Money Spent based on Retention Status](https://github.com/shamit24/RetentionSQLProject/blob/4b3c37be1f4d8ba5e98b67be0a07430327d62bdd/Money%20Spent%20based%20on%20Retention.png)

Link to sheets: https://docs.google.com/spreadsheets/d/1yBpE0wsp_haIWUrJQE11YqygO7IgR72tJ_zqr3O-q8A/edit?usp=sharing

### Do players with rolling 30-day retention come from specific regions?

```sql

SELECT 
  location, 
  COUNT(player_id) as total_retained_players 
FROM 
  (
    SELECT 
      days_info.joining_day, 
      days_info.player_id as pid 
    FROM 
      (
        SELECT 
          players.player_id, 
          players.joined as joining_day, 
          lastplay.lastplay_day 
        FROM 
          `junoproject1.mobilegamedata.player_info` players 
          JOIN (
            SELECT 
              player_id as player_lastday, 
              MAX(day) as lastplay_day 
            FROM 
              `junoproject1.mobilegamedata.matches_info` 
            GROUP BY 
              1
          ) as lastplay ON players.player_id = lastplay.player_lastday
      ) as days_info 
    WHERE 
      days_info.lastplay_day - days_info.joining_day > 30
  ) AS r_info 
  JOIN `junoproject1.mobilegamedata.player_info` as p_info ON p_info.player_id = r_info.pid 
GROUP BY 
  1 
ORDER BY 
  2 DESC
  ```

* We again join the `mathces_info` and `player_info` tables together 
* Filter out only the retained players using the `WHERE` clause
* Then we `COUNT` the `player_id` as they are filtered to be only the retained players
* Finally we `GROUP BY` location to output our final table
  
  ![Retained Players based on Location](https://github.com/shamit24/RetentionSQLProject/blob/4b3c37be1f4d8ba5e98b67be0a07430327d62bdd/Retained%20players%20per%20location.png)
  
  According to our analysis, the largest amount of retained players come from South America. After visualizing the data, we can see that the difference in the amnount of retained players per location is very minimal.
 
 [Interactive Map](https://docs.google.com/spreadsheets/d/e/2PACX-1vSNjUCcF6Tu5XPjHwf5luYinXfNRdObWCAqkxyV_zYmRthILOC4msEeEpiQzr5JE6DCYTBMrlt5PpV2/pubchart?oid=540534240&format=interactive)
 
 Link to Sheets: https://docs.google.com/spreadsheets/d/1bQrKVcCmgUtd7rskSz9TLCGxcUnru79BT3437bXkkWE/edit?usp=sharing
 
