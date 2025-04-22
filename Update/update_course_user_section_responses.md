create function update_course_user_section_responses(course_ids integer[]) returns void
    language plpgsql
as
$$
DECLARE
    course_id INTEGER;
    test_id INTEGER;
    last_processed_time TIMESTAMP WITH TIME ZONE;
    total_processed_count INTEGER := 0;
    processed_count INTEGER;
    seq_offset INTEGER;
BEGIN
    IF course_ids IS NULL OR array_length(course_ids, 1) IS NULL THEN
        RAISE NOTICE 'No course IDs provided.';
        RETURN;
    END IF;

    FOREACH course_id IN ARRAY course_ids LOOP
        BEGIN
            SELECT MAX("updatedAt")
            INTO last_processed_time
            FROM "CourseUserSectionResponses"
            WHERE "courseId" = course_id;

            IF last_processed_time IS NULL THEN
                last_processed_time := '1970-01-01 00:00:00+00'::TIMESTAMP WITH TIME ZONE;
            END IF;

            RAISE NOTICE 'Processing course ID % with last_processed_time: %', course_id, last_processed_time;

            FOR test_id IN
                SELECT DISTINCT ct."testId"
                FROM "CourseTest" ct
                JOIN "Test" t ON ct."testId" = t.id
                WHERE ct."courseId" = course_id
                AND t."sections" IS NOT NULL
                AND json_array_length(t."sections") > 0
            LOOP
                RAISE NOTICE 'Processing test ID %', test_id;

                -- Calculate seqNum offset for this test
                SELECT MIN("seqNum") - 1 INTO seq_offset
                FROM "TestQuestion"
                WHERE "testId" = test_id;

                IF seq_offset IS NULL THEN
                    RAISE NOTICE 'No questions found for test ID %, skipping', test_id;
                    CONTINUE;
                END IF;

                RAISE NOTICE 'SeqNum offset for test ID %: %', test_id, seq_offset;

                CREATE TEMP TABLE temp_section_ranges (
                    "testId" INTEGER,
                    section_name TEXT,
                    start_question INTEGER,
                    end_question INTEGER
                ) ON COMMIT DROP;

                INSERT INTO temp_section_ranges
                SELECT
                    t.id AS "testId",
                    TRIM((elem->>0)::TEXT) AS section_name,
                    (elem->>1)::INTEGER AS start_question,
                    COALESCE(
                        LEAD((elem->>1)::INTEGER) OVER (PARTITION BY t.id ORDER BY (elem->>1)::INTEGER),
                        t."numQuestions" + 1
                    ) - 1 AS end_question
                FROM "Test" t
                CROSS JOIN LATERAL json_array_elements(t."sections") AS elem
                WHERE t.id = test_id
                AND (elem->>0) IS NOT NULL
                AND TRIM((elem->>0)::TEXT) != '';

                IF NOT EXISTS (SELECT 1 FROM temp_section_ranges) THEN
                    RAISE NOTICE 'No valid sections for test ID %, skipping', test_id;
                    DROP TABLE temp_section_ranges;
                    CONTINUE;
                END IF;

                -- Process new answers for this test
                INSERT INTO "CourseUserSectionResponses" (
                    "courseId",
                    "userId",
                    "testId",
                    "sectionName",
                    "responseCount",
                    "totalQuestionsInSection",
                    "updatedAt"
                )
                SELECT
                    ct."courseId",
                    ta."userId",
                    ct."testId",
                    sr.section_name,
                    COUNT(DISTINCT a."questionId")::INTEGER AS "responseCount",
                    (sr.end_question - sr.start_question + 1)::INTEGER AS "totalQuestionsInSection",
                    CURRENT_TIMESTAMP
                FROM "CourseTest" ct
                JOIN "TestAttempt" ta ON ct."testId" = ta."testId"
                JOIN "Answer" a ON ta.id = a."testAttemptId"
                JOIN "TestQuestion" tq ON a."questionId" = tq."questionId" AND tq."testId" = ct."testId"
                JOIN temp_section_ranges sr ON ct."testId" = sr."testId"
                WHERE ct."courseId" = course_id
                AND ct."testId" = test_id
                AND a."createdAt" > last_processed_time
                AND (tq."seqNum" - seq_offset) >= sr.start_question
                AND (tq."seqNum" - seq_offset) <= sr.end_question
                GROUP BY
                    ct."courseId",
                    ta."userId",
                    ct."testId",
                    sr.section_name,
                    sr.start_question,
                    sr.end_question
                ON CONFLICT ON CONSTRAINT "CourseUserSectionResp_courseId_userId_testId_sect_key"
                DO UPDATE SET
                    "responseCount" = EXCLUDED."responseCount",
                    "totalQuestionsInSection" = EXCLUDED."totalQuestionsInSection",
                    "updatedAt" = CURRENT_TIMESTAMP;

                GET DIAGNOSTICS processed_count = ROW_COUNT;
                total_processed_count := total_processed_count + processed_count;
                RAISE NOTICE 'Inserted/Updated % rows for test ID % with new answers', processed_count, test_id;

                -- Fallback: Process all data if no new answers
                IF processed_count = 0 THEN
                    RAISE NOTICE 'No new answers, processing all data for test ID %', test_id;
                    INSERT INTO "CourseUserSectionResponses" (
                        "courseId",
                        "userId",
                        "testId",
                        "sectionName",
                        "responseCount",
                        "totalQuestionsInSection",
                        "updatedAt"
                    )
                    SELECT
                        ct."courseId",
                        ta."userId",
                        ct."testId",
                        sr.section_name,
                        COUNT(DISTINCT a."questionId")::INTEGER AS "responseCount",
                        (sr.end_question - sr.start_question + 1)::INTEGER AS "totalQuestionsInSection",
                        CURRENT_TIMESTAMP
                    FROM "CourseTest" ct
                    JOIN "TestAttempt" ta ON ct."testId" = ta."testId"
                    JOIN "Answer" a ON ta.id = a."testAttemptId"
                    JOIN "TestQuestion" tq ON a."questionId" = tq."questionId" AND tq."testId" = ct."testId"
                    JOIN temp_section_ranges sr ON ct."testId" = sr."testId"
                    WHERE ct."courseId" = course_id
                    AND ct."testId" = test_id
                    AND (tq."seqNum" - seq_offset) >= sr.start_question
                    AND (tq."seqNum" - seq_offset) <= sr.end_question
                    GROUP BY
                        ct."courseId",
                        ta."userId",
                        ct."testId",
                        sr.section_name,
                        sr.start_question,
                        sr.end_question
                    ON CONFLICT ON CONSTRAINT "CourseUserSectionResp_courseId_userId_testId_sect_key"
                    DO UPDATE SET
                        "responseCount" = EXCLUDED."responseCount",
                        "totalQuestionsInSection" = EXCLUDED."totalQuestionsInSection",
                        "updatedAt" = CURRENT_TIMESTAMP;

                    GET DIAGNOSTICS processed_count = ROW_COUNT;
                    total_processed_count := total_processed_count + processed_count;
                    RAISE NOTICE 'Inserted/Updated % rows for all data in test ID %', processed_count, test_id;
                END IF;

                DROP TABLE temp_section_ranges;
            END LOOP;

            RAISE NOTICE 'Processed course ID %: % total rows updated', course_id, total_processed_count;

        EXCEPTION WHEN OTHERS THEN
            RAISE NOTICE 'Error for course ID %: %', course_id, SQLERRM;
            DROP TABLE IF EXISTS temp_section_ranges;
        END;
    END LOOP;

    RAISE NOTICE 'Completed for course IDs: %', course_ids;
END;
$$;

alter function update_course_user_section_responses(integer[]) owner to learner;
