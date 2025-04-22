create function update_course_question_counts_for_courses(p_course_ids integer[]) returns integer
    language plpgsql
as
$$
DECLARE
    affected_rows INTEGER := 0;
    total_affected_rows INTEGER := 0;
    batch_size INTEGER := 1000;
    min_course_id INTEGER;
    max_course_id INTEGER;
    current_min_id INTEGER;
BEGIN
    RAISE NOTICE 'Starting update_course_question_counts_for_courses at % with courseIds: %', CURRENT_TIMESTAMP, p_course_ids;

    -- If the input array is empty, return 0
    IF array_length(p_course_ids, 1) IS NULL THEN
        RAISE NOTICE 'No courseIds provided';
        RETURN 0;
    END IF;

    -- Get the range of courseIds to process from the input array
    SELECT MIN(course_id), MAX(course_id)
    INTO min_course_id, max_course_id
    FROM unnest(p_course_ids) AS course_id;

    IF min_course_id IS NULL THEN
        RAISE NOTICE 'No valid courseIds to process';
        RETURN 0;
    END IF;

    current_min_id := min_course_id;

    -- Process in batches
    WHILE current_min_id <= max_course_id LOOP
        INSERT INTO "CourseQuestionDetail" (
            "courseId",
            "questionsAll",
            "questionsDifficult",
            "questionsEasy",
            "questionsMedium",
            "createdAt",
            "updatedAt"
        )
        SELECT
            ct."courseId",
            COALESCE(SUM(CASE WHEN st."difficultyLevel" = 'all' THEN 1 ELSE 0 END), 0) AS "questionsAll",
            COALESCE(SUM(CASE WHEN st."difficultyLevel" = 'difficult' THEN 1 ELSE 0 END), 0) AS "questionsDifficult",
            COALESCE(SUM(CASE WHEN st."difficultyLevel" = 'easy' THEN 1 ELSE 0 END), 0) AS "questionsEasy",
            COALESCE(SUM(CASE WHEN st."difficultyLevel" = 'medium' THEN 1 ELSE 0 END), 0) AS "questionsMedium",
            CURRENT_TIMESTAMP,
            CURRENT_TIMESTAMP
        FROM "CourseTest" ct
        JOIN "Test" t ON ct."testId" = t.id
            AND (t."expiryAt" IS NULL OR t."expiryAt" > CURRENT_TIMESTAMP)
        JOIN "SubTest" st ON st."testId" = t.id
        JOIN "Test" sub_t ON st."subtestId" = sub_t.id
            AND (sub_t."expiryAt" IS NULL OR sub_t."expiryAt" > CURRENT_TIMESTAMP)
        JOIN "TestQuestion" tq ON tq."testId" = sub_t.id
        WHERE ct."courseId" = ANY(p_course_ids)
          AND ct."courseId" BETWEEN current_min_id AND (current_min_id + batch_size - 1)
        GROUP BY ct."courseId"
        ON CONFLICT ("courseId") DO UPDATE
        SET
            "questionsAll" = EXCLUDED."questionsAll",
            "questionsDifficult" = EXCLUDED."questionsDifficult",
            "questionsEasy" = EXCLUDED."questionsEasy",
            "questionsMedium" = EXCLUDED."questionsMedium",
            "updatedAt" = CURRENT_TIMESTAMP;

        -- Get the number of affected rows in this batch
        GET DIAGNOSTICS affected_rows = ROW_COUNT;
        total_affected_rows := total_affected_rows + affected_rows;

        -- Move to the next batch
        current_min_id := current_min_id + batch_size;
    END LOOP;

    RAISE NOTICE 'Finished update_course_question_counts_for_courses at %, total affected rows: %', CURRENT_TIMESTAMP, total_affected_rows;

    RETURN total_affected_rows;
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Error in update_course_question_counts_for_courses: %', SQLERRM;
        RETURN -1;
END;
$$;

alter function update_course_question_counts_for_courses(integer[]) owner to learner;
