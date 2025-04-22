CREATE FUNCTION getachievementdetails(userid integer, subject text)
    RETURNS TABLE(current_level integer, next_level integer, questions_till_next_level integer, achieved_at timestamp without time zone, achievement_id integer, progress_message text)
    LANGUAGE plpgsql
AS
$$
DECLARE
    current_achievement    RECORD;
    next_achievement       RECORD;
    total_questions_solved INT;
BEGIN
    -- Log the start of the function
    RAISE NOTICE 'Starting GetAchievementDetails for userId: %, subject: %', userId, subject;
    
    -- Calculate total questions solved in this subject by the user
    SELECT SUM(a."currentLevelUnit")
    INTO total_questions_solved
    FROM "UserAchievement" ua
             JOIN "Achievement" a ON ua."achievementId" = a."id"
    WHERE ua."userId" = userId
      AND a.description ILIKE '%' || subject || '%';
    
    -- Get the highest non-upgraded achievement for the specified subject
    SELECT ua."achievementId", a."level" AS current_level, ua."achievedAt", a."currentLevelUnit", a."nextLevelUnit"
    INTO current_achievement
    FROM "UserAchievement" ua
             JOIN "Achievement" a ON ua."achievementId" = a."id"
    WHERE ua."userId" = userId
      AND a.description ILIKE '%' || subject || '%'
      AND ua."upgraded" = false
    ORDER BY a."currentLevelUnit" DESC
    LIMIT 1;
    
    -- If no current achievement is found, set default values and find the first level
    IF NOT FOUND THEN
        current_level := NULL;
        achieved_at := NULL;
        achievement_id := NULL;
        
        -- Find the first level achievement
        SELECT a."level" AS next_level, a."currentLevelUnit", a."id"
        INTO next_achievement
        FROM "Achievement" a
        WHERE a.description ILIKE '%' || subject || '%'
        ORDER BY a."currentLevelUnit"
        LIMIT 1;
        
        IF FOUND THEN
            next_level := next_achievement.next_level;
            questions_till_next_level := next_achievement."currentLevelUnit";
            progress_message :=
                    'No achievements found for the specified subject. Starting at level ' || next_level || '.';
        ELSE
            next_level := NULL;
            questions_till_next_level := NULL;
            progress_message := 'No achievements found for the specified subject and no next levels available.';
        END IF;
        RETURN NEXT;
    END IF;
    
    -- If a current achievement was found, proceed with finding the next level
    current_level := current_achievement.current_level;
    achieved_at := current_achievement."achievedAt";
    achievement_id := current_achievement."achievementId";
    
    -- Find the next level achievement, if available
    SELECT a."level" AS next_level, a."currentLevelUnit", a."id"
    INTO next_achievement
    FROM "Achievement" a
    WHERE a."currentLevelUnit" > current_achievement."currentLevelUnit"
      AND a.description ILIKE '%' || subject || '%'
    ORDER BY a."currentLevelUnit"
    LIMIT 1;
    
    -- Check if the next achievement was found
    IF FOUND THEN
        next_level := next_achievement.next_level;
        questions_till_next_level := next_achievement."currentLevelUnit";
        progress_message := 'Current achievements found. Next level: ' || next_level || '.';
    ELSE
        next_level := NULL;
        questions_till_next_level := NULL;
        progress_message := 'Current achievements found but no next levels available.';
    END IF;
    
    RETURN NEXT;
END;
$$;

ALTER FUNCTION getachievementdetails(integer, text) OWNER TO neetprep_rw;
