create function update_user_streak_data(p_user_id integer, p_streak_count integer, p_last_streak_date date, p_last_streak_updated date) returns integer
    language plpgsql
as
$$
BEGIN
    -- Update or insert the streak data in UserStreakData
    IF EXISTS (SELECT 1 FROM "UserStreakData" WHERE "userId" = p_user_id) THEN
        UPDATE "UserStreakData"
        SET "streakCount" = p_streak_count,
            "lastStreakDate" = p_last_streak_date,
            "lastStreakUpdated" = p_last_streak_updated,
            "updatedAt" = CURRENT_TIMESTAMP
        WHERE "userId" = p_user_id;
        RAISE NOTICE 'Streak data updated for user ID %. New streak: %', p_user_id, p_streak_count;
    ELSE
        INSERT INTO "UserStreakData" ("userId", "streakCount", "lastStreakDate", "lastStreakUpdated", "createdAt", "updatedAt")
        VALUES (p_user_id, p_streak_count, p_last_streak_date, p_last_streak_updated, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
        RAISE NOTICE 'Streak data inserted for user ID %. Streak: %', p_user_id, p_streak_count;
    END IF;

    -- Return the streak count
    RETURN p_streak_count;
END;
$$;

alter function update_user_streak_data(integer, integer, date, date) owner to learner;
