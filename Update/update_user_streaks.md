create function update_user_streaks() returns void
    language plpgsql
as
$$
DECLARE
    v_user_id INT; -- User ID variable
    v_streak INT; -- Streak count
    v_last_streak_date DATE; -- Last streak update date
    v_check_date DATE; -- The current date being checked in the loop
    v_ans_count INT; -- Total answers in the last 7 days
    v_correct_ans_count INT; -- Total correct answers in the last 7 days
BEGIN
    -- Step 1: Delete old streak data for the last 7 days to ensure a clean start
    DELETE FROM "FocussedUserStreakData"
    WHERE "lastStreakUpdated" >= DATE(CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata') - INTERVAL '7 days';

    RAISE NOTICE 'Deleted streak data for the last 7 days.';

    -- Step 2: Iterate from 7 days ago to yesterday (using IST)
    FOR v_check_date IN
        (SELECT * FROM GENERATE_SERIES(
            DATE(CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata') - INTERVAL '7 days', 
            DATE(CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata') - INTERVAL '2 day', 
            '1 day' 
        )) LOOP  -- Use GENERATE_SERIES directly in SELECT and loop over it
        RAISE NOTICE 'Checking date: %', v_check_date; -- Print the current check date

        -- Iterate over users who were active on the current check date
        FOR v_user_id IN (
            SELECT DISTINCT "userId"
            FROM "DailyUserEvent"
            WHERE "event" = 'Question'
            AND "eventCount" > 100
            AND "eventDate" = v_check_date
        ) LOOP
            RAISE NOTICE 'Processing user: %', v_user_id; -- Print the user being processed
            
            -- Fetch current streak data for each user
            SELECT "streakCount", "lastStreakUpdated"
            INTO v_streak, v_last_streak_date
            FROM "FocussedUserStreakData"
            WHERE "userId" = v_user_id
            LIMIT 1;

            -- Default values if no record exists
            IF v_streak IS NULL THEN
                v_streak := 0;
                v_last_streak_date := (v_check_date) - INTERVAL '2 years'; -- Set a default old date
                RAISE NOTICE 'No existing streak data found, initializing streak count to 0 and date to 2 years ago';
            ELSE
                RAISE NOTICE 'Current streak for user %: % (last updated on %)', v_user_id, v_streak, v_last_streak_date;
            END IF;

            -- Calculate answers in the last 7 days (for all 'Question' events) in IST
            SELECT 
                COALESCE(SUM("eventCount"), 0) AS ans_count
            INTO v_ans_count
            FROM "DailyUserEvent"
            WHERE "userId" = v_user_id
            AND "event" = 'Question'
            AND "eventDate" >= (v_check_date) - INTERVAL '7 days'
            AND "eventDate" <= (v_check_date);
            RAISE NOTICE 'Total answers in the last 7 days for user %: %', v_user_id, v_ans_count;

            -- Calculate correct answers in the last 7 days (for 'CorrectQuestion' events) in IST
            SELECT 
                COALESCE(SUM("eventCount"), 0) AS correct_ans_count
            INTO v_correct_ans_count
            FROM "DailyUserEvent"
            WHERE "userId" = v_user_id
            AND "event" = 'CorrectQuestion'
            AND "eventDate" >= (v_check_date) - INTERVAL '7 days'
            AND "eventDate" <= (v_check_date);
            RAISE NOTICE 'Total correct answers in the last 7 days for user %: %', v_user_id, v_correct_ans_count;

            -- Handling activity detection for the user
            IF v_ans_count > 0 THEN
                RAISE NOTICE 'Activity found for user %: new streak count = %', v_user_id, v_streak;

                -- Determine if there's a break in the streak (based on IST)
                IF v_last_streak_date = v_check_date - INTERVAL '1 day' THEN
                    -- Continuous streak (since last update was the previous day)
                    v_streak := v_streak + 1;
                    RAISE NOTICE 'Streak continued for user %: new streak count = %', v_user_id, v_streak;
                ELSE
                    -- Streak broken, reset to 1
                    v_streak := 1;
                    RAISE NOTICE 'Streak broken for user %: resetting streak to 1', v_user_id;
                END IF;
            ELSE
                RAISE NOTICE 'Activity not found for user %: new streak count = %', v_user_id, v_streak;
                v_streak := 0;
            END IF;

            -- Update or insert streak data
            IF EXISTS (SELECT 1 FROM "FocussedUserStreakData" WHERE "userId" = v_user_id) THEN
                -- Update the streak record if it already exists
                UPDATE "FocussedUserStreakData"
                SET "streakCount" = v_streak,
                    "lastStreakUpdated" = v_check_date,
                    "ansCount7Days" = v_ans_count,
                    "correctAnsCount7Days" = v_correct_ans_count,
                    "updatedAt" = CURRENT_TIMESTAMP
                WHERE "userId" = v_user_id;
                RAISE NOTICE 'Updated streak for user % on %: streak = %, answers = %, correct answers = %', v_user_id, v_check_date, v_streak, v_ans_count, v_correct_ans_count;
            ELSE
                -- Insert a new streak record for the user if it doesn't exist
                INSERT INTO "FocussedUserStreakData" ("userId", "streakCount", "lastStreakUpdated", "ansCount7Days", "correctAnsCount7Days", "createdAt", "updatedAt")
                VALUES (v_user_id, v_streak, v_check_date, v_ans_count, v_correct_ans_count, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
                RAISE NOTICE 'Inserted new streak record for user % on %: streak = %, answers = %, correct answers = %', v_user_id, v_check_date, v_streak, v_ans_count, v_correct_ans_count;
            END IF;
        END LOOP;
    END LOOP;
END $$;

alter function update_user_streaks() owner to neetprep_rw;
