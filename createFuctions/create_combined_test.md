# SQL Function: create_combined_test

```sql
create function create_combined_test(p_test_pages jsonb, p_user_id integer) returns integer
    language plpgsql
as
$$
DECLARE
    test_id INTEGER;
    num_questions INTEGER := 0;
    duration_in_min INTEGER;
    chapter_test INTEGER;
    pages JSONB;
    page TEXT;
    max_questions INTEGER := 200; -- Maximum allowed questions in a test
BEGIN
    -- Step 1: Create a single test entry
    INSERT INTO "Test" ("name", "description", "userId", "sections", "numQuestions", "durationInMin", "syllabus")
    VALUES (
        'Combined Custom Test',
        'Generated test for multiple chapters and pages',
        p_user_id,
        '[]'::JSONB,
        0,
        0,
        'Syllabus for selected chapters'
    )
    RETURNING "id" INTO test_id;

    -- Step 2: Loop through each chapterId and its associated pages in the JSONB input
    FOR chapter_test, pages IN
        SELECT key::INTEGER, value
        FROM jsonb_each(p_test_pages)
    LOOP
        -- Loop through each page in the pages array
        FOR page IN SELECT value FROM jsonb_array_elements_text(pages) LOOP
            -- Stop adding questions if the limit is reached
            IF num_questions >= max_questions THEN
                EXIT;
            END IF;

            -- Insert questions from the specific chapter and page
            INSERT INTO "TestQuestion" ("testId", "questionId")
            SELECT
                test_id,
                "questionId"
            FROM test_questions_by_sections tqs
            WHERE tqs."testId" = chapter_test
            AND tqs."section" = page
            LIMIT (max_questions - num_questions) -- Add only up to the remaining limit
            ON CONFLICT ("testId", "questionId") DO NOTHING; -- Skip duplicate questionId for the same testId

            -- Update the current count of questions
            SELECT COUNT(*) INTO num_questions
            FROM "TestQuestion"
            WHERE "testId" = test_id;

            -- Stop adding further questions if limit is reached
            IF num_questions >= max_questions THEN
                EXIT;
            END IF;
        END LOOP;

        -- Exit the outer loop if the limit is reached
        IF num_questions >= max_questions THEN
            EXIT;
        END IF;
    END LOOP;

    -- Step 3: Update the test with the number of questions and duration
    duration_in_min := num_questions * 2; -- Assuming 2 minutes per question as a default duration

    UPDATE "Test"
    SET
        "numQuestions" = num_questions,
        "durationInMin" = duration_in_min
    WHERE "id" = test_id;

    -- Step 4: Log the test creation
    INSERT INTO "UserDpp" ("testId", "userId", "subTopics")
    VALUES (test_id, p_user_id, p_test_pages);

    -- Return the created test ID
    RETURN test_id;
END;
$$;

alter function create_combined_test(jsonb, integer) owner to neetprep_rw;
