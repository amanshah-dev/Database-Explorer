-- Create function to calculate topic mastery proficiency for a user
create function calculate_topic_master_proficiency(userid integer, requiredtopicscount integer) returns boolean
    language plpgsql
as
$$
DECLARE
    proficient_topics_count INT;  -- Variable to store the number of proficient topics
BEGIN
    -- Count the number of topics where the user has 85% or higher proficiency
    SELECT COUNT(*)
    INTO proficient_topics_count
    FROM "UserTopicAnswers" uta
    WHERE "userId" = userId
      AND (SELECT 100.0 * COUNT(is_correct) FILTER (WHERE is_correct) / GREATEST(COUNT(*), 1)
           FROM unnest("isCorrectArray") is_correct) >= 85;

    -- Return true if the user has proficiency in the required number of topics
    RETURN proficient_topics_count >= requiredTopicsCount;
END;
$$;

-- Alter function owner
alter function calculate_topic_master_proficiency(integer, integer) owner to neetprep_rw;
