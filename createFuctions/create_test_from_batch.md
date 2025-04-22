CREATE FUNCTION create_test_from_batch(temp_test_id INTEGER) RETURNS INTEGER
    LANGUAGE plpgsql
AS
$$
DECLARE
    test_id INTEGER;
    batch_test_name TEXT;
    seq_num INTEGER = 0;
    question RECORD;
    section_data JSONB;
    num_questions INTEGER;
    duration_in_min INTEGER;
    section_record RECORD;
    cumulative_sec_qs_count INTEGER = 1;
BEGIN
    -- Get the BatchTest name
    SELECT bt."name" INTO batch_test_name
    FROM "BatchTest" bt
    WHERE bt."tempTestId" = temp_test_id;

    -- Create a new test in the Test table
    INSERT INTO "Test" ("name", "description", "sections", "numQuestions", "durationInMin", "syllabus", "free", "startedAt", "expiryAt")
    VALUES (batch_test_name, batch_test_name, '[]'::JSONB, 0, 0, '', true, current_timestamp, current_timestamp + interval '2 weeks')
    RETURNING "id" INTO test_id;

    -- Process each section separately
    FOR section_record IN
        SELECT DISTINCT "section"
        FROM "TempTestQuestion"
        WHERE "tempTestId" = temp_test_id AND "deleted" = false
        ORDER BY "section"
    LOOP
        -- Insert questions for the current section in a randomized order
        FOR question IN
            SELECT "questionId"
            FROM "TempTestQuestion"
            WHERE "tempTestId" = temp_test_id AND "section" = section_record."section" AND "deleted" = false
            ORDER BY random()
        LOOP
            seq_num := seq_num + 1;
            INSERT INTO "TestQuestion" ("testId", "questionId", "seqNum", "createdAt", "updatedAt")
            VALUES (test_id, question."questionId", seq_num, NOW(), NOW());
        END LOOP;

        -- Add the section to the section_data JSONB
        section_data := coalesce(section_data, '[]'::JSONB) || jsonb_build_array(jsonb_build_array(section_record."section", cumulative_sec_qs_count));
        cumulative_sec_qs_count := cumulative_sec_qs_count + (SELECT COUNT(*) FROM "TempTestQuestion" WHERE "tempTestId" = temp_test_id AND "section" = section_record."section" AND "deleted" = false);
    END LOOP;

    -- Calculate the number of questions and duration for the test
    SELECT COUNT(*), COUNT(*) INTO num_questions, duration_in_min
    FROM "TestQuestion"
    WHERE "testId" = test_id;

    -- Update the Test table with sections, numQuestions, and durationInMin
    UPDATE "Test"
    SET "sections" = section_data, "numQuestions" = num_questions, "durationInMin" = duration_in_min
    WHERE "id" = test_id;

    -- Update the BatchTest table to associate the created test with the batch
    UPDATE "BatchTest"
    SET "testId" = test_id
    WHERE "tempTestId" = temp_test_id;

    RETURN test_id;
END;
$$;

ALTER FUNCTION create_test_from_batch(INTEGER) OWNER TO neetprep_rw;
