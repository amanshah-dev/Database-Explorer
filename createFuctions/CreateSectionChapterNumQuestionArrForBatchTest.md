CREATE FUNCTION "CreateSectionChapterNumQuestionArrForBatchTest"(batchtestid INTEGER) 
    RETURNS chapter_section_questions[]
    LANGUAGE plpgsql
AS
$$
DECLARE
    total_num_questions INTEGER;
    pattern_section RECORD;
    chapter_index INTEGER;
    chapter_section_count INTEGER;
    scaled_question_count INTEGER;
    results_array chapter_section_questions[] := ARRAY[]::chapter_section_questions[]; -- Explicitly typed empty array initialization

    -- Variables to hold data from the BatchTest and BatchTestChapter tables
    testPatternId INTEGER;
    chapterId INTEGER;
    numQuestions INTEGER;
    chapter_record RECORD;
BEGIN
    -- Retrieve the examPatternId from BatchTest
    SELECT "testPatternId" INTO testPatternId FROM "BatchTest" WHERE "id" = batchtestid;

    -- Loop through each chapter and calculate question distribution based on sections
    FOR chapter_record IN SELECT "chapterId", "numQuestions" FROM "BatchTestChapter" WHERE "batchTestId" = batchtestid LOOP
        -- Get the chapterId and numQuestions from the current record
        chapterId := chapter_record."chapterId";
        numQuestions := chapter_record."numQuestions";

        -- Get the total count of sections for this chapter
        SELECT COUNT(*) INTO chapter_section_count FROM "ChapterSection" WHERE "chapterId" = chapterId;

        -- Prevent division by zero by checking if chapter_section_count is greater than zero
        IF chapter_section_count > 0 THEN
            -- Calculate number of questions per section for this chapter
            scaled_question_count := CEILING(CAST(numQuestions AS FLOAT) / chapter_section_count);
        ELSE
            -- Handle case with no sections
            scaled_question_count := 0;
        END IF;

        -- Loop through sections associated with this chapter to allocate questions
        FOR pattern_section IN SELECT "section" FROM "ChapterSection" WHERE "chapterId" = chapterId LOOP
            -- Check for sections and add to array
            IF chapter_section_count > 0 THEN
                results_array := array_append(results_array, ROW(chapterId, pattern_section."section", scaled_question_count)::chapter_section_questions);
            END IF;
        END LOOP;

        -- Optionally handle chapters with no sections if needed
        IF chapter_section_count = 0 THEN
            -- Optionally manage cases with no sections
        END IF;
    END LOOP;

    RETURN "ScaleSectionQuestions"(results_array, testPatternId);
END;
$$;

ALTER FUNCTION "CreateSectionChapterNumQuestionArrForBatchTest"(INTEGER) OWNER TO neetprep_rw;
