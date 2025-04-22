create function compute_user_streak(p_user_id integer, p_event_count integer)
    returns TABLE(streak_count integer, last_streak_date date, last_streak_updated date, needs_update boolean)
    language plpgsql
as
$$
DECLARE
    v_streak INT := 0;
    v_last_streak_date DATE;
    v_last_streak_updated DATE;
    v_check_date DATE;
    v_last_missing_date DATE := NULL;
    v_has_activity_today BOOLEAN := FALSE;
    v_current_date DATE := CURRENT_DATE;
    v_needs_update BOOLEAN := TRUE;
BEGIN
    -- Retrieve the last streak updated date, last streak date, and streak count from UserStreakData
    SELECT "lastStreakUpdated"::DATE, COALESCE("lastStreakDate"::DATE, "lastStreakUpdated"::DATE), COALESCE("streakCount", 0)
    INTO v_last_streak_updated, v_last_streak_date, v_streak
    FROM "UserStreakData"
    WHERE "userId" = p_user_id
    LIMIT 1;

    RAISE NOTICE 'Last streak updated date: %, Last streak date: %, Initial streak count: %', v_last_streak_updated, v_last_streak_date, v_streak;

    -- If streak was already updated today, return the current streak count with no update needed
    IF v_last_streak_updated = v_current_date THEN
        RAISE NOTICE 'Streak already updated today. Current streak: %', v_streak;
        v_needs_update := FALSE;
        RETURN QUERY SELECT v_streak, v_last_streak_date, v_last_streak_updated, v_needs_update;
        RETURN;
    END IF;

    -- Check if the streak was updated yesterday
    IF v_last_streak_updated = v_current_date - INTERVAL '1 day' THEN
        -- No detailed check required, use yesterday as the check date
        RAISE NOTICE 'Detailed check not required. Using yesterday as check date.'; 
        v_check_date := v_current_date - INTERVAL '1 day';
    ELSE
        -- If the streak was not updated yesterday, perform a detailed check
        RAISE NOTICE 'Detailed check for streak update required.';

        -- If there's no record of lastStreakUpdated, use 2 years ago as the starting point
        IF v_last_streak_updated IS NULL THEN
            RAISE NOTICE 'No last streak date found. Using 2 years ago as starting point.';
            v_check_date := v_current_date - INTERVAL '2 years';
        ELSE
            -- Use the lastStreakUpdated as the starting point
            v_check_date := v_last_streak_updated;
        END IF;

        -- Query to find the last missing event date in DailyUserEvent
        WITH DateSeries AS (
            SELECT GENERATE_SERIES(v_check_date, v_current_date - INTERVAL '1 day', '1 day')::DATE AS DateValue
        )
        SELECT MAX(DateSeries.DateValue) AS LastMissingDate
        INTO v_last_missing_date
        FROM DateSeries
        LEFT JOIN "DailyUserEvent" DUE 
            ON DateSeries.DateValue = DUE."eventDate"
            AND DUE."userId" = p_user_id
            AND DUE."event" = 'Question'
            AND DUE."eventCount" >= p_event_count
        WHERE DUE."eventDate" IS NULL;

        -- If no missing date was found in DailyUserEvent, check Answer table
        IF v_last_missing_date IS NOT NULL THEN
            -- For each potentially missing date, check the Answer table as a fallback
            WITH DateSeries AS (
                SELECT GENERATE_SERIES(v_check_date, v_last_missing_date, '1 day')::DATE AS DateValue
            ),
            AnswerDates AS (
                SELECT 
                    DATE(("createdAt" AT TIME ZONE 'Asia/Kolkata')) AS AnswerDate,
                    COUNT(DISTINCT "questionId") AS QuestionCount
                FROM "Answer"
                WHERE "userId" = p_user_id
                  AND DATE(("createdAt" AT TIME ZONE 'Asia/Kolkata')) BETWEEN v_check_date AND v_last_missing_date
                  AND "createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ
                GROUP BY AnswerDate
                HAVING COUNT(DISTINCT "questionId") >= p_event_count
            )
            SELECT MAX(DateSeries.DateValue) AS LastMissingDate
            INTO v_last_missing_date
            FROM DateSeries
            LEFT JOIN AnswerDates
                ON DateSeries.DateValue = AnswerDates.AnswerDate
            WHERE AnswerDates.AnswerDate IS NULL;
        END IF;

    END IF;

    -- If no missing date was found, set v_last_missing_date to v_check_date
    IF v_last_missing_date IS NULL THEN
        v_last_missing_date := v_check_date;
    END IF;

    -- Check if user has activity today
    SELECT EXISTS (
        SELECT 1
        FROM "DailyUserEvent"
        WHERE "userId" = p_user_id
        AND "event" = 'Question'
        AND "eventCount" >= p_event_count
        AND "eventDate" = v_current_date
    ) INTO v_has_activity_today;

    -- If no activity found in DailyUserEvent, check the Answer table
    IF NOT v_has_activity_today THEN
        SELECT EXISTS (
            SELECT 1
            FROM "Answer"
            WHERE "userId" = p_user_id
            AND DATE(("createdAt" AT TIME ZONE 'Asia/Kolkata')) = v_current_date
            AND "createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ
            GROUP BY "userId", DATE(("createdAt" AT TIME ZONE 'Asia/Kolkata'))
            HAVING COUNT(DISTINCT "questionId") >= p_event_count
        ) INTO v_has_activity_today;
    END IF;

    -- Determine if update is needed based on the calculation
    DECLARE 
        v_old_streak INT := v_streak;
        v_old_last_streak_date DATE := v_last_streak_date;
        v_old_last_streak_updated DATE := v_last_streak_updated;
    BEGIN
        -- Adjust streak calculation based on activity after the last missing date
        IF v_last_missing_date = v_check_date THEN
            IF v_has_activity_today THEN
                -- Include today's activity
                v_streak := v_streak + (v_current_date - v_last_missing_date);
                v_last_streak_updated := v_current_date;
                v_last_streak_date := v_current_date;
            ELSE
                -- No activity today, streak up to yesterday
                v_streak := v_streak + (v_current_date - v_last_missing_date) - 1;
                v_last_streak_updated := v_current_date - INTERVAL '1 day';
                v_last_streak_date := v_current_date - INTERVAL '1 day';
            END IF;
        ELSE
            -- Break in the streak, reset the streak count
            IF v_has_activity_today THEN
                v_streak := CURRENT_DATE - v_last_missing_date;
                v_last_streak_updated := CURRENT_DATE;  -- Set last updated date to today
                v_last_streak_date := CURRENT_DATE;     -- Set last streak date to today
            ELSE
                v_streak := (CURRENT_DATE - v_last_missing_date) - 1;
                v_last_streak_updated := CURRENT_DATE - INTERVAL '1 day';  -- Set last updated date to yesterday
                v_last_streak_date := CURRENT_DATE - INTERVAL '1 day';     -- Set last streak date to yesterday
            END IF;
        END IF;
    END;

    -- Return the computed streak, last streak date, last streak update date, and whether an update is needed
    RETURN QUERY SELECT v_streak, v_last_streak_date, v_last_streak_updated, v_needs_update;
END;
$$;

alter function compute_user_streak(integer, integer) owner to learner;
