create function update_yesterday_user_streak() returns void
    language plpgsql
as
$$
DECLARE
    v_user_id INT; -- User ID variable
    v_last_streak_date DATE; -- Last streak update date
    v_streak INT; -- Streak count
    v_ans_count INT; -- Total answers in the last 7 days
    v_correct_ans_count INT; -- Total correct answers in the last 7 days
    v_check_date DATE := DATE(CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata') - INTERVAL '1 day'; -- Date for checking activity (Indian time)
    v_seven_days_ago DATE := DATE(CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata') - INTERVAL '7 days'; -- Seven days ago date
BEGIN
    -- Output the checking date and 7 days ago date for debugging
    RAISE NOTICE 'Checking date: %, Seven days ago: %', v_check_date, v_seven_days_ago;

    -- Iterate over users active yesterday (Indian Standard Time)
    FOR v_user_id IN (
        SELECT DISTINCT "userId"
        FROM "DailyUserEvent"
        WHERE "event" = 'Question'
        AND "eventCount" > 100
        AND "eventDate" = v_check_date
    ) LOOP
        RAISE NOTICE 'Processing user_id: %', v_user_id; -- Output the user_id being processed

        -- Fetch current streak data
        SELECT COALESCE("streakCount", 0), COALESCE("lastStreakUpdated", v_check_date - INTERVAL '2 years')
        INTO v_streak, v_last_streak_date
        FROM "FocussedUserStreakData"
        WHERE "userId" = v_user_id
        LIMIT 1;

        -- Output current streak and last streak update date for debugging
        RAISE NOTICE 'Current streak for user %: %, Last streak updated: %', v_user_id, v_streak, v_last_streak_date;

        -- Calculate answers in the last 7 days (for all 'Question' events) in Indian time
        SELECT 
            SUM("eventCount") AS ans_count
        INTO v_ans_count
        FROM "DailyUserEvent"
        WHERE "userId" = v_user_id
        AND "event" = 'Question'
        AND "eventDate" >= v_seven_days_ago
        AND "eventDate" <= v_check_date;
        
        -- Calculate answers in the last 7 days (for all 'Question' events) in Indian time
        SELECT
            SUM(CASE WHEN "event" = 'CorrectQuestion' THEN "eventCount" ELSE 0 END) AS correct_ans_count
        INTO v_correct_ans_count
        FROM "DailyUserEvent"
        WHERE "userId" = v_user_id
        AND "event" = 'CorrectQuestion'
        AND "eventDate" >= v_seven_days_ago
        AND "eventDate" <= v_check_date;

        -- Output the total answers and correct answers count for debugging
        RAISE NOTICE 'Total answers for user % in the last 7 days: %, Correct answers: %', v_user_id, v_ans_count, v_correct_ans_count;

        -- Determine if there's a break in the streak (based on Indian time)
        IF v_last_streak_date = v_check_date - INTERVAL '1 day' THEN
            -- Continuous streak
            v_streak := v_streak + 1;
            RAISE NOTICE 'Streak is continuous for user %: New streak count is %', v_user_id, v_streak;
        ELSE
            -- Streak broken, reset to 1
            v_streak := 1;
            RAISE NOTICE 'Streak is broken for user %: Resetting streak count to %', v_user_id, v_streak;
        END IF;

        -- Update or insert streak data
        IF EXISTS (SELECT 1 FROM "FocussedUserStreakData" WHERE "userId" = v_user_id) THEN
            -- Update the streak record if it already exists
            RAISE NOTICE 'Updating streak for user %', v_user_id;
            UPDATE "FocussedUserStreakData"
            SET "streakCount" = v_streak,
                "lastStreakUpdated" = v_check_date,
                "ansCount7Days" = v_ans_count,
                "correctAnsCount7Days" = v_correct_ans_count,
                "updatedAt" = CURRENT_TIMESTAMP
            WHERE "userId" = v_user_id;
        ELSE
            -- Insert a new streak record for the user if it doesn't exist
            RAISE NOTICE 'Inserting new streak record for user %', v_user_id;
            INSERT INTO "FocussedUserStreakData" ("userId", "streakCount", "lastStreakUpdated", "ansCount7Days", "correctAnsCount7Days", "createdAt", "updatedAt")
            VALUES (v_user_id, v_streak, v_check_date, v_ans_count, v_correct_ans_count, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
        END IF;

    END LOOP;

    RAISE NOTICE 'Streak update process completed.'; -- End of the function
END $$;

alter function update_yesterday_user_streak() owner to neetprep_rw;
