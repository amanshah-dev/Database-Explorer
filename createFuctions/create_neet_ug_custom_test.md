CREATE FUNCTION create_neet_ug_custom_test(chapterids integer[], testpatternid integer, userid integer) RETURNS integer
    LANGUAGE plpgsql
AS
$$
DECLARE
    testId INTEGER;
    generated_questions RECORD;
    quesCounts INTEGER[];
    numQuestions INTEGER;
    durationInMin INTEGER;
    temp_sections JSONB;
    seqNum INTEGER = 0;
    chap_sec_qs chapter_section_questions[];
    section_record RECORD;
    syllabus TEXT;
    cumulative_sec_qs_count INTEGER = 1;
BEGIN
    -- Generate a sample array for quesCounts with each entry as 10
    quesCounts := ARRAY(SELECT 10 FROM generate_series(1, array_length(chapterIds, 1)));

    -- Create syllabus based on chapter ids
    SELECT string_agg("name"::text, ', ') INTO syllabus FROM "Topic" WHERE "id" = ANY(chapterIds);

    -- Create a new test in the Test table
    INSERT INTO "Test" ("name", "description", "userId", "sections", "numQuestions", "durationInMin", "syllabus")
    VALUES ('Custom Practice Test - ' ||  TO_CHAR(CURRENT_DATE, 'DD-Mon'), 'Custom Practice Test - ' ||  TO_CHAR(CURRENT_DATE, 'DD-Mon'), userId, '[]'::JSONB, 0, 0, syllabus)
    RETURNING "id" INTO testId;

    -- Generate the chapter_section_questions array
    chap_sec_qs := "CreateSectionChapterNumQuestionArr"(testPatternId, chapterIds, quesCounts);

    -- Generate questions using "GenerateQuestionsFromQuesBank"
    INSERT INTO "TestQuestion" ("testId", "questionId", "seqNum")
    SELECT testId, q.question_id,
           (tps."seqId" * 1000 + q.row_num) AS "seqNum"
    FROM (SELECT *, row_number() OVER (ORDER BY random()) AS row_num 
          FROM "GenerateQuestionsFromQuesBank"(testPatternId, chap_sec_qs, ('{"function_name": "remove_user_attempted_questions", "args": {"p_userId": ' || userId::TEXT || '}}')::jsonb)) q
    JOIN "TestPatternSection" tps ON q."section_name" = tps."section"
    WHERE tps."testPatternId" = testPatternId;

    -- Calculate the number of questions and duration for the test
    SELECT COUNT(*), COUNT(*) INTO numQuestions, durationInMin
    FROM "TestQuestion"
    WHERE "testId" = testId;

    -- Generate sections JSONB data
    temp_sections := '[]'::JSONB;
    FOR section_record IN
        SELECT DISTINCT ON (tps."section") tps."section", tps."numQuestions" AS numQuestions, tps."maxAttempts" AS maxAttempts
        FROM unnest(chap_sec_qs) AS chap_sec
        JOIN "TestPatternSection" tps ON chap_sec.section_name = tps."section"
        WHERE tps."testPatternId" = testPatternId
        ORDER BY tps."section", tps."seqId"
    LOOP
        IF section_record.maxAttempts IS NULL THEN
            temp_sections := temp_sections || jsonb_build_array(jsonb_build_array(section_record.section, cumulative_sec_qs_count));
        ELSE
            temp_sections := temp_sections || jsonb_build_array(jsonb_build_array(section_record.section, cumulative_sec_qs_count, section_record.maxAttempts));
        END IF;
        cumulative_sec_qs_count := cumulative_sec_qs_count + section_record.numQuestions;
    END LOOP;

    -- Update the Test table with sections, numQuestions, and durationInMin
    UPDATE "Test"
    SET "sections" = temp_sections, "numQuestions" = numQuestions, "durationInMin" = durationInMin
    WHERE "id" = testId;

    -- Add entry into UserDPP table for this test to show in user entries
    INSERT INTO "UserDpp" ("testId", "userId") VALUES (testId, userId);

    RETURN testId;
END;
$$;

ALTER FUNCTION create_neet_ug_custom_test(integer[], integer, integer) OWNER TO neetprep_rw;

