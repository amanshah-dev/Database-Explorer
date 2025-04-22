-- Create function to calculate topic master proficiency for multiple users
create function calculate_topic_master_proficients(userids bigint[], requiredtopicscount integer) returns bigint[]
    language plpgsql
as
$$
DECLARE
    proficient_users BIGINT[] := Array []::BIGINT[]; -- Array to store the result
BEGIN
    -- Fetch the users with proficiency in the required number of topics
    SELECT ARRAY_AGG("userId")
    INTO proficient_users
    FROM (SELECT "userId"
          FROM "UserTopicAnswers" uta,
               LATERAL (
                   SELECT 100.0 * COUNT(is_correct) FILTER (WHERE is_correct) / 
                          GREATEST(COUNT(*), 1) AS correctness_percentage
                   FROM unnest(uta."isCorrectArray") is_correct
                   ) correct_subquery
          WHERE "userId" = ANY (userIds)
            AND correctness_percentage >= 85
          GROUP BY "userId"
          HAVING COUNT(*) > requiredTopicsCount -- Only return users with proficient topics count greater than requiredTopicsCount
         ) subquery;

    -- Return the array of userIds who have mastered the required number of topics
    RETURN proficient_users;
END;
$$;

-- Alter function owner
alter function calculate_topic_master_proficients(bigint[], integer) owner to neetprep_rw;
