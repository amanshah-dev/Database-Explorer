-- Create function to calculate the user's answer streak within a date range
create function calculate_user_answer_streak(p_user_id integer, p_latest_date date, p_earliest_date date DEFAULT '1970-01-01'::date) returns integer
    strict
    language sql
as
$$
WITH answer_events AS (
    SELECT 
        event_date AS "eventDate",
        LAG(event_date) OVER (ORDER BY event_date) AS "prevDate"
    FROM (
        SELECT 
            ("createdAt" AT TIME ZONE 'Asia/Kolkata')::DATE AS event_date
        FROM "Answer"
        WHERE "userId" = p_user_id
    ) AS answers
    WHERE event_date <= p_latest_date
    AND event_date > p_earliest_date
),
streak_groups AS (
    SELECT
        "eventDate",
        SUM(CASE WHEN "eventDate" - "prevDate" > 1 THEN 1 ELSE 0 END) 
            OVER (ORDER BY "eventDate") AS "streakId"
    FROM answer_events
),
current_streaks AS (
    SELECT 
        COUNT(distinct("eventDate")) AS "streakLength",
        MAX("eventDate") AS "maxDate"
    FROM streak_groups
    GROUP BY "streakId"
    HAVING MAX("eventDate") = p_latest_date
)
SELECT COALESCE( (SELECT "streakLength" FROM current_streaks), 0 )
$$;

-- Alter function owner
alter function calculate_user_answer_streak(integer, date, date) owner to learner;
