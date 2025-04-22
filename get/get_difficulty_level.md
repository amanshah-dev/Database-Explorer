CREATE FUNCTION get_difficulty_level(correct_answer_count bigint, incorrect_answer_count bigint) 
    RETURNS text
    LANGUAGE plpgsql
AS
$$
DECLARE
    correct_percentage NUMERIC;
    difficulty_level TEXT;
BEGIN
    -- Get the correct percentage based on the counts of correct and incorrect answers
    SELECT get_correct_percentage(correct_answer_count, incorrect_answer_count) INTO correct_percentage;

    -- If the correct percentage is NULL, return NULL
    IF correct_percentage IS NULL THEN 
        RETURN NULL;
    END IF;

    -- Determine the difficulty level based on the correct percentage
    IF correct_percentage >= 70.0::NUMERIC THEN 
        RETURN 'easy'::TEXT;
    ELSEIF correct_percentage >= 50.0::NUMERIC AND correct_percentage < 70.0::NUMERIC THEN 
        RETURN 'medium'::TEXT;
    ELSEIF correct_percentage >= 0::NUMERIC AND correct_percentage < 50.0::NUMERIC THEN 
        RETURN 'difficult'::TEXT;
    ELSE
        RETURN NULL;
    END IF;
END;
$$;

ALTER FUNCTION get_difficulty_level(bigint, bigint) OWNER TO learner;
