CREATE FUNCTION createsectionchapternumquestionarr(exampatternid INTEGER, chapterids INTEGER[], quescounts INTEGER[])
    RETURNS TABLE(chapter_id INTEGER, section_name TEXT, num_questions INTEGER)
    LANGUAGE plpgsql
AS
$$
DECLARE
    total_num_questions INTEGER;
    pattern_section RECORD;
    chapter_index INTEGER;
    total_section_count INTEGER;
    chapter_section_count INTEGER;
    scaled_question_count INTEGER;
    chapter_question_distribution INTEGER[];
BEGIN
    -- Retrieve the total number of questions for the given examPatternId from TestPattern
    SELECT "numQuestions" INTO total_num_questions FROM "TestPattern" WHERE "id" = exampatternid;

    -- Initialize the array to hold the distribution of questions for each chapter and section
    chapter_question_distribution := ARRAY[]::INTEGER[];

    -- Loop through each chapter and calculate question distribution based on sections
    FOR chapter_index IN 1..array_length(chapterIds, 1) LOOP
        -- Get the total count of sections for this chapter
        SELECT COUNT(*) INTO chapter_section_count FROM "ChapterSection" WHERE "chapterId" = chapterIds[chapter_index];

        -- Calculate number of questions per section for this chapter
        scaled_question_count := quesCounts[chapter_index] / chapter_section_count;

        -- Loop through sections associated with this chapter to allocate questions
        FOR pattern_section IN SELECT "section", "seqId" FROM "ChapterSection" WHERE "chapterId" = chapterIds[chapter_index] LOOP
            -- Add to the results
            chapter_question_distribution := array_append(chapter_question_distribution, scaled_question_count);

            -- Output results for each section of each chapter
            RETURN NEXT;
            chapter_id := chapterIds[chapter_index];
            section_name := pattern_section."section";
            num_questions := scaled_question_count;
        END LOOP;
    END LOOP;

    -- Adjust the distribution if the total doesn't match
    IF SUM(chapter_question_distribution) <> total_num_questions THEN
        -- Logic to adjust the distribution to match total_num_questions
    END IF;

    RETURN;
END;
$$;

ALTER FUNCTION createsectionchapternumquestionarr(integer, integer[], integer[]) OWNER TO neetprep_rw;
