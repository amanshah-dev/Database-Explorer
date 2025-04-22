create function update_user_streak(p_user_id integer, p_event_count integer) returns integer
    language plpgsql
as
$$
DECLARE
    v_streak INT := 0;  -- Initialize the streak counter
    v_prev_date DATE := NULL;  -- Variable to track the previous date
    v_event_date DATE;  -- Scalar variable to hold the event date
    v_last_streak_date DATE;  -- The last date the streak was updated
    v_current_ist_date DATE;  -- The current date in 'Asia/Kolkata' timezone
    v_check_date DATE;  -- Date to check streak from
    v_last_missing_date DATE := NULL;  -- Last date where event was missing
    v_has_activity_today BOOLEAN := FALSE;  -- Flag to check if user has activity today
    v_last_streak_update DATE;  -- Variable to store the updated last streak date
    dateUpper TIMESTAMP;
    dateLower TIMESTAMP;
BEGIN
    -- Retrieve the last streak updated date and streak count from UserStreakData
    SELECT COALESCE("lastStreakDate", date(timezone('Asia/Kolkata', "lastStreakUpdated"))), COALESCE("streakCount", 0)
    INTO v_last_streak_date, v_streak
    FROM "UserStreakData"
    WHERE "userId" = p_user_id
    LIMIT 1;

    RAISE NOTICE 'Last streak updated date: %, Initial streak count: %', v_last_streak_date, v_streak;

    -- If v_streak is NULL, set it to 0
    IF v_streak IS NULL THEN
        v_streak := 0;
    END IF;

    SELECT date(timezone('Asia/Kolkata', now())) INTO v_current_ist_date;

    -- If the streak was already updated today, return the current streak count
    IF v_last_streak_date = v_current_ist_date THEN
        RETURN v_streak;
    ELSE
        -- temp fix start: Call part of FUNCTION RefreshUserStreak(userid) for now
       SELECT v_current_ist_date - interval '2 day' into dateLower;
       SELECT v_current_ist_date + interval '1 day' into dateUpper;
       RAISE NOTICE 'starting time: %, ending time: %', dateLower, dateUpper;
        insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount")
          select
            "userId", 'Question', ("createdAt" at time zone 'Asia/Kolkata')::DATE as "eventDate",
            count(distinct("questionId"))
          from
            "Answer"
          where
            "createdAt" >= dateLower and
            "createdAt" < dateUpper and
            "userId" = p_user_id
            and "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ -- FORCE USE OF PARTIAL INDEX ON ANSWER
          group by
            "userId", "eventDate" on conflict ("userId", "event", "eventDate")
            where
              "courseId" is null do nothing;
        -- temp fix end: Call part of FUNCTION RefreshUserStreak(userid) for now

        -- If the streak was updated yesterday, set the check date to yesterday
        IF v_last_streak_date = v_current_ist_date - INTERVAL '1 day' THEN
            -- No detailed check required, use yesterday as the check date
            RAISE NOTICE 'Detailed check not required. Using yesterday as check date.';
            v_check_date := v_current_ist_date - INTERVAL '1 day';
        ELSE
            -- If the streak was not updated yesterday, perform a detailed check
            RAISE NOTICE 'Detailed check for streak update required.';
            -- If there's no record of lastStreakUpdated, use 2 years ago as the starting point
            IF v_last_streak_date IS NULL THEN
                RAISE NOTICE 'No last streak date found. Using 2 years ago as starting point.';

                v_check_date := v_current_ist_date - INTERVAL '2 years';
                v_last_streak_date := v_current_ist_date - INTERVAL '2 years';
                RAISE NOTICE 'Using date: % as the starting point for streak calculation.', v_check_date;

            ELSE
                -- Use the lastStreakUpdated as the starting point
                v_check_date := v_last_streak_date;
                RAISE NOTICE 'Using last streak updated date: %', v_last_streak_date;
            END IF;

            -- Query to find the last missing event date
            WITH DateSeries AS (
                SELECT
                    GENERATE_SERIES(
                        v_check_date,
                        v_current_ist_date - INTERVAL '1 day',
                        '1 day'
                    )::DATE AS DateValue
            )
            SELECT
                MAX(DateSeries.DateValue) AS LastMissingDate
            INTO v_last_missing_date
            FROM
                DateSeries
            LEFT JOIN
                "DailyUserEvent" DUE ON DateSeries.DateValue = DUE."eventDate"
                AND DUE."userId" = p_user_id
                AND DUE."event" = 'Question'
                AND DUE."eventCount" >= p_event_count
            WHERE
                DUE."eventDate" IS NULL;

            RAISE NOTICE 'Last missing date: %', v_last_missing_date;
        END IF;

        -- If no missing date was found, set v_last_missing_date to v_check_date
        IF v_last_missing_date IS NULL THEN
            v_last_missing_date := v_check_date;
            RAISE NOTICE 'No missing dates found, using v_check_date as last missing date: %', v_last_missing_date;
        END IF;

        -- Check if user is having any activity today
        SELECT EXISTS (
            SELECT 1
            FROM "DailyUserEvent"
            WHERE "userId" = p_user_id
            AND "event" = 'Question'
            AND "eventCount" >= p_event_count
            AND "eventDate" = v_current_ist_date
        ) INTO v_has_activity_today;

        -- Adjust streak calculation based on today's activity and missing date
        IF v_last_missing_date = v_check_date THEN
            IF v_has_activity_today THEN
                -- No break in the streak, include today's activity
                v_streak := v_streak + (v_current_ist_date - v_last_missing_date);
                v_last_streak_update := v_current_ist_date;  -- Set last updated date to today
                RAISE NOTICE 'Streak incremented with today''s activity. Current streak: % days', v_streak;
            ELSE
                -- No activity today, streak up to yesterday
                v_streak := v_streak + (v_current_ist_date - v_last_missing_date) - 1;
                v_last_streak_update := v_current_ist_date - INTERVAL '1 day';  -- Set last updated date to yesterday
                RAISE NOTICE 'Streak incremented without today''s activity. Current streak: % days', v_streak;
            END IF;
        ELSE
            -- Break in the streak, reset the streak count
            IF v_has_activity_today THEN
                -- User has activity today, reset the streak count but include today's activity
                v_streak := v_current_ist_date - v_last_missing_date;
                v_last_streak_update := v_current_ist_date;  -- Set last updated date to today
                RAISE NOTICE 'Streak reset but includes today''s activity. Current streak: % days', v_streak;
            ELSE
                -- No activity today, reset the streak count
                v_streak := (v_current_ist_date - v_last_missing_date) - 1;
                v_last_streak_update := v_current_ist_date - INTERVAL '1 day';  -- Set last updated date to yesterday
                RAISE NOTICE 'Streak reset. Current streak: % days', v_streak;
            END IF;
        END IF;

        -- Update or insert the streak data in UserStreakData
        IF EXISTS (SELECT 1 FROM "UserStreakData" WHERE "userId" = p_user_id) THEN
            IF v_last_streak_date = v_current_ist_date - INTERVAL '1 day' AND NOT v_has_activity_today THEN
                RAISE NOTICE 'Streak data updated yesterday, no activity today %.', p_user_id;
            ELSE
                -- Update the existing streak record
                UPDATE "UserStreakData"
                SET "streakCount" = v_streak,
                    "lastStreakUpdated" = v_last_streak_update,
                    "lastStreakDate" = v_last_streak_update,  -- Populate new column
                    "updatedAt" = CURRENT_TIMESTAMP
                WHERE "userId" = p_user_id;
                RAISE NOTICE 'Streak data updated for user ID %.', p_user_id;
            END IF;
        ELSE
            -- Insert a new streak record
            INSERT INTO "UserStreakData" ("userId", "streakCount", "lastStreakUpdated", "lastStreakDate", "createdAt", "updatedAt")
            VALUES (p_user_id, v_streak, v_last_streak_update, v_last_streak_update, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
            RAISE NOTICE 'Streak data inserted for user ID %.', p_user_id;
        END IF;
    END IF;

    -- Return the updated streak count
    RETURN v_streak;
END;
$$;

alter function update_user_streak(integer, integer) owner to learner;
