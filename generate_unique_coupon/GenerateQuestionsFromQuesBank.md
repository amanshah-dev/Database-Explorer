CREATE FUNCTION "GenerateQuestionsFromQuesBank"(testpatternid INTEGER, scaled_sections chapter_section_questions[])
    RETURNS TABLE(chapter_id INTEGER, section_name section_type, question_id INTEGER)
    LANGUAGE plpgsql
AS
$$
DECLARE
    row chapter_section_questions;
    total_questions_needed INTEGER;
    questions_remaining INTEGER;
    selected_question RECORD;
    test_ids INTEGER[];
    test_index INTEGER;
BEGIN
    -- Create a temporary table to store results
    CREATE TEMP TABLE temp_result (
        chapter_id INTEGER,
        section_name section_type,
        question_id INTEGER
    ) ON COMMIT DROP;

    -- Loop through each row in the scaled_sections array
    FOREACH row IN ARRAY scaled_sections LOOP
        -- Initialize the remaining number of questions to be selected
        total_questions_needed := row.num_questions;
        questions_remaining := total_questions_needed;

        -- Retrieve test IDs from QuesBankTest for the given chapter and section
        test_ids := get_chapter_section_test_ids(row.chapter_id, row.section_name, testPatternId);

        -- Log if test_ids is null or empty
        /*IF test_ids IS NULL OR array_length(test_ids, 1) IS NULL THEN
            RAISE NOTICE 'No test IDs found for chapter_id: %, section_name: %', row.chapter_id, row.section_name;
            CONTINUE; -- Skip to the next row if no test IDs found
        END IF;*/

        -- Loop through the test IDs to fetch the required number of questions
        FOR test_index IN 1..array_length(test_ids, 1) LOOP
            -- Select the required number of questions randomly
            FOR selected_question IN
                WITH duplicate_questions AS (
                    -- Get all duplicate questions where questionId2 is always greater than questionId1 for the same test
                    SELECT
                        "dq"."questionId2" AS "duplicateId"
                    FROM
                        "TestQuestion" tq1
                    JOIN
                        "DuplicateQuestion" dq
                        ON "tq1"."testId" = test_ids[test_index]
                        AND ("dq"."questionId1" = "tq1"."questionId" OR "dq"."questionId2" = "tq1"."questionId")
                    JOIN
                        "TestQuestion" tq2
                        ON "tq2"."testId" = test_ids[test_index]
                        AND ("dq"."questionId1" = "tq2"."questionId" OR "dq"."questionId2" = "tq2"."questionId")
                        AND "tq2"."questionId" <> "tq1"."questionId"
                ),
                temp_result_duplicates AS (
                    -- Get all duplicate questions from the temp_result where questionId2 is always greater than questionId1
                    SELECT
                        "dq"."questionId2" AS "duplicateId"
                    FROM
                        temp_result tr1
                    JOIN
                        "DuplicateQuestion" dq
                        ON ("dq"."questionId1" = "tr1"."question_id" OR "dq"."questionId2" = "tr1"."question_id")
                    JOIN
                        temp_result tr2
                        ON ("dq"."questionId1" = "tr2"."question_id" OR "dq"."questionId2" = "tr2"."question_id")
                        AND "tr2"."question_id" <> "tr1"."question_id"
                )
                SELECT "questionId"
                FROM "TestQuestion", "Question"
                WHERE "testId" = test_ids[test_index]
                    AND "Question"."id" = "TestQuestion"."questionId"
                    AND "Question"."deleted" = false
                    AND "Question"."published" = true
                    AND "Question"."type" = 'MCQ-SO'
                    AND "questionId"
                    NOT IN (
                        SELECT temp_result.question_id FROM temp_result
                        UNION SELECT "duplicateId" FROM duplicate_questions
                        UNION SELECT "duplicateId" FROM temp_result_duplicates)
                ORDER BY RANDOM()
                LIMIT questions_remaining
            LOOP
                -- Insert the selected question into the temporary table
                INSERT INTO temp_result (chapter_id, section_name, question_id)
                VALUES (row.chapter_id, row.section_name, selected_question."questionId");

                -- Decrease the number of questions remaining
                questions_remaining := questions_remaining - 1;

                -- Exit the loop if we have enough questions
                EXIT WHEN questions_remaining <= 0;
            END LOOP;

            -- Exit the loop if we have enough questions
            EXIT WHEN questions_remaining <= 0;
        END LOOP;

        -- Handle the case where not enough questions were found
        IF questions_remaining > 0 THEN
            RAISE NOTICE 'Not enough questions found for chapter % and section %', row.chapter_id, row.section_name;
        END IF;
    END LOOP;

    -- Return the results from the temporary table
    RETURN QUERY SELECT * FROM temp_result;

    -- Clean up the temporary table
    DROP TABLE temp_result;

    RETURN;
END;
$$;

ALTER FUNCTION "GenerateQuestionsFromQuesBank"(integer, chapter_section_questions[]) OWNER TO neetprep_rw;
