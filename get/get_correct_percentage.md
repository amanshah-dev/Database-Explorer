CREATE FUNCTION get_correct_percentage(correct_answer_count BIGINT, incorrect_answer_count BIGINT) 
    RETURNS NUMERIC 
    LANGUAGE plpgsql
AS
$$
DECLARE
    correct_percentage NUMERIC;
BEGIN
    -- If the total number of answers is less than or equal to 10, return NULL
    IF correct_answer_count + incorrect_answer_count <= 10 THEN 
        RETURN NULL;
    END IF;
    
    -- Calculate the correct percentage based on the number of correct and incorrect answers
    RETURN ((correct_answer_count::NUMERIC * 1.0) / (correct_answer_count::NUMERIC + incorrect_answer_count::NUMERIC))::NUMERIC * 100;
END;
$$;

ALTER FUNCTION get_correct_percentage(BIGINT, BIGINT) OWNER TO learner;
