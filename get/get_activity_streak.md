CREATE FUNCTION get_activity_streak(p_user_id INTEGER) 
    RETURNS INTEGER
    LANGUAGE plpgsql
AS
$$
DECLARE
    v_streak INT; -- Variable to hold the streak count
BEGIN
    -- Retrieve the streak count from the UserStreakData table
    SELECT 
        COALESCE("streakCount", 0)
    INTO 
        v_streak
    FROM 
        "UserStreakData"
    WHERE 
        "userId" = p_user_id;

    -- Return the streak count, default to 0 if no data exists
    RETURN COALESCE(v_streak, 0);
END;
$$;

ALTER FUNCTION get_activity_streak(integer) OWNER TO learner;
