create procedure update_streakmaster(IN achievementcategoryid bigint, IN userids bigint[] DEFAULT ARRAY[]::bigint[])
    language plpgsql
as
$$
DECLARE
    max_days           INT := 90;
    user_streaks       RECORD;
    achievement_record RECORD;
    eligibleUsers      INTEGER[];
BEGIN
    IF NOT CheckStreak(userIds, 7) THEN
        RETURN;
    END IF;
    -- Calculate the streaks for multiple users
    WITH date_series AS (SELECT generate_series(CURRENT_DATE - max_days + 1, CURRENT_DATE, '1 day'::interval) AS date),
         user_events AS (SELECT "userId", "eventDate"
                         FROM "DailyUserEvent"
                         WHERE "userId" = ANY (userIds) -- Handle multiple userIds
                           AND "eventDate" >= CURRENT_DATE - max_days + 1
                           AND "event" LIKE '%Question%'
                           AND "eventCount" > 0),
         streak_data AS (SELECT user_events."userId",
                                date_series.date,
                                SUM(CASE WHEN user_events."eventDate" IS NOT NULL THEN 1 ELSE 0 END)
                                OVER (PARTITION BY user_events."userId" ORDER BY date_series.date DESC) AS running_count
                         FROM date_series
                                  CROSS JOIN (SELECT DISTINCT "userId" FROM user_events) users -- Ensure we handle each user
                                  LEFT JOIN user_events ON date_series.date = user_events."eventDate" AND
                                                           users."userId" = user_events."userId"),
         streaks AS (SELECT "userId", COUNT(*) AS streak_length
                     FROM streak_data
                     WHERE running_count = date - CURRENT_DATE + 1
                     GROUP BY "userId", date)
-- Fetch the maximum streak length for each user
    SELECT "userId", COALESCE(MAX(streak_length), 0) AS user_streak
    INTO user_streaks -- Store the result into the array or table
    FROM streaks
    GROUP BY "userId";


    -- Fetch all relevant achievements in one query
    FOR achievement_record IN (SELECT a.id,
                                      a."achievementCategoryId",
                                      a."currentLevelUnit" AS days
                               FROM "Achievement" a
                               WHERE a."achievementCategoryId" = achievementCategoryId
                               ORDER BY days DESC)
        LOOP
            SELECT array_agg(user_streaks."userId")
            INTO eligibleUsers
            FROM user_streaks
            WHERE user_streak >= achievement_record.days;
            CALL insert_user_achievements(eligibleUsers, achievement_record);
        END LOOP;
END;
$$;

alter procedure update_streakmaster(bigint, bigint[]) owner to neetprep_rw;
