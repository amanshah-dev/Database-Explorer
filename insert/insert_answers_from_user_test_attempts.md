create function insert_answers_from_user_test_attempts(p_user_id integer, p_test_ids integer[] DEFAULT NULL::integer[], p_hours_ago integer DEFAULT 48)
    returns TABLE(inserted_count integer, skipped_count integer)
    language plpgsql
as
$$
DECLARE
    v_test_ids INTEGER[];
    total_inserted INTEGER := 0;
    total_skipped INTEGER := 0;
BEGIN
    -- Use provided test_ids or retrieve from cache
    IF p_test_ids IS NULL THEN
        v_test_ids := ARRAY(SELECT test_id FROM get_score_booster_test_ids(p_user_id));
    ELSE
        v_test_ids := p_test_ids;
    END IF;

    -- Temporary table to store answers to be inserted
    CREATE TEMPORARY TABLE temp_answers (
                                            "questionId" INTEGER,
                                            "userAnswer" INTEGER,
                                            "userId" INTEGER,
                                            "testAttemptId" INTEGER,
                                            "durationInSec" INTEGER,
                                            "createdAt" TIMESTAMP WITH TIME ZONE,
                                            "updatedAt" TIMESTAMP WITH TIME ZONE
    ) ON COMMIT DROP;

    -- Extract answers from TestAttempt for the user's recent score booster test attempts
    INSERT INTO temp_answers (
        "questionId",
        "userAnswer",
        "userId",
        "testAttemptId",
        "durationInSec",
        "createdAt",
        "updatedAt"
    )
    SELECT
        (key::TEXT)::INTEGER AS "questionId",  -- Safer casting
        CASE
            WHEN jsonb_typeof(value) = 'number' THEN (value::TEXT)::INTEGER
            WHEN jsonb_typeof(value) = 'string' AND (value #>> '{}') ~ '^-?\d+$' THEN (value #>> '{}')::INTEGER
        END AS "userAnswer",
        ta."userId",
        ta."id" AS "testAttemptId",
        COALESCE(
                (ta."userQuestionWiseDurationInSec"->>(key::text))::INTEGER,
                FLOOR(ta."elapsedDurationInSec" / (SELECT COUNT(*) FROM jsonb_object_keys(ta."userAnswers"::jsonb)))) AS "durationInSec",
                ta."updatedAt" AS "createdAt",
                ta."updatedAt" AS "updatedAt"
                FROM
                "TestAttempt" ta,
                jsonb_each(ta."userAnswers"::jsonb)
                WHERE
                ta."userId" = p_user_id
                    AND ta."createdAt" > (NOW() - interval '1 hour' * p_hours_ago)
                    AND ta."testId" = ANY(v_test_ids);

    -- Insert into Answer table with ON CONFLICT DO NOTHING
    WITH inserted_answers AS (
        INSERT INTO "Answer" (
                              "questionId",
                              "userAnswer",
                              "userId",
                              "testAttemptId",
                              "durationInSec",
                              "createdAt",
                              "updatedAt"
            )
            SELECT
                "questionId",
                "userAnswer",
                "userId",
                "testAttemptId",
                "durationInSec",
                "createdAt",
                "updatedAt"
            FROM temp_answers ta
            ON CONFLICT ("userId", "questionId", "testAttemptId") DO NOTHING
            RETURNING 1
    )
    SELECT
        COUNT(*) INTO total_inserted
    FROM inserted_answers;

    -- Calculate skipped records
    total_skipped := (SELECT COUNT(*) FROM temp_answers) - total_inserted;

    RETURN QUERY
        SELECT total_inserted, total_skipped;
END;
$$;

alter function insert_answers_from_user_test_attempts(integer, integer[], integer) owner to learner;
