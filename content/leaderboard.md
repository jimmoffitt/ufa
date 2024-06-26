# Building a leaderboard 

The FlappyTinybird project features a personalized leaderboard with a summary of top scorers, presented along side your own scores and recent game history. This leaderboard is an example of User-Facing Analytics (UFA). 

The project includes two implementation of the leaderboard. We'll start with the more simple version and the SQL it uses to rank the top scorers. Then we will review how the leraderboard is implemented in the production game. This more complex version combines the top scorers and the user's own scores and is based on Materialized Views to increase query performance.

## **api_leaderboard** Pipe

### filter_data Node

The `filter_data` Node returns 'score' events from the `events_api` Data Source.

```sql
SELECT name AS player_id,
       session_id
FROM events_api
WHERE type = 'score'
```

### aggregrate Node

The `aggregate` Node counts the 'score' events and returns the top 10 scores. 

```sql
SELECT player_id, session_id, count() AS score
FROM filter_data
GROUP BY player_id, session_id
ORDER BY score DESC
LIMIT 10
```


# Materialized View details

First, the following Data Source is generated by a Pipe.

```bash
# Data Source created from Pipe 'mat_player_stats'

SCHEMA >
    `event` String,
    `player_id` String,
    `session_id` String,
    `scores` AggregateFunction(countIf, UInt8),
    `purchases` AggregateFunction(countIf, UInt8),
    `start_ts` AggregateFunction(min, DateTime64(3)),
    `end_ts` AggregateFunction(max, DateTime64(3))

ENGINE "AggregatingMergeTree"
ENGINE_SORTING_KEY "event, player_id, session_id"
```

### rank_games

This query returns all of the game results, ranked by scores (in individual sessions). 

```sql
SELECT
 ROW_NUMBER() OVER (ORDER BY total_score DESC, t) AS rank,
 player_id,
 session_id,
 countMerge(scores) AS total_score,
 maxMerge(end_ts) AS t
FROM mv_player_stats
GROUP BY player_id, session_id
ORDER BY rank
```

### last_game

The query returns the max total score for a player indicated with the `player_param` query parameter. 

```sql
SELECT
 argMax(rank, t) AS rank,
 player_id,
 argMax(session_id, t) AS session_id,
 argMax(total_score, t) AS total_score
FROM rank_games
WHERE
 player_id
 = {{ String(player_param, 'player1', description="Player to filter on", required=True) }}
GROUP BY player_id
```

### endpoint

This query combines the top ten list with the score of the player. 

```sql
SELECT *
FROM
 (
     SELECT rank, player_id, session_id, total_score
     FROM rank_games
     WHERE
         (player_id, session_id)
         NOT IN (SELECT player_id, session_id FROM last_game)
     LIMIT 9
     UNION ALL
     SELECT rank, player_id, session_id, total_score
     FROM last_game
 )
ORDER BY rank
```

