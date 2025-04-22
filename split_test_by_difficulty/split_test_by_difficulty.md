create function split_test_by_difficulty(original_test_id integer) returns void
    language plpgsql
as
$$
DECLARE
    easy_test_id INTEGER;
    medium_test_id INTEGER;
    difficult_test_id INTEGER;
BEGIN
    -- Step 1: Create new tests with correctly computed sections
    INSERT INTO "Test" (
        "name", "description", "instructions", "durationInMin", "positiveMarks", "negativeMarks", 
        "ownerId", "ownerType", "creatorId", "createdAt", "updatedAt", "numQuestions", "syllabus", 
        "free", "sections", "importUrl", "resultMsgHtml", "showAnswer", "year", "exam", "pdfURL", 
        "userId", "scholarship", "reviewAt", "discussionEnd", "seqId", "allowPracticeMode", "chapterId", "lockSection"
    )
    SELECT 
        CONCAT("name", ' - Easy'), "description", "instructions", "durationInMin", "positiveMarks", "negativeMarks", 
        "ownerId", "ownerType", "creatorId", NOW(), NOW(), 0, "syllabus", "free", 
        (
            WITH section_question_counts AS (
                SELECT 
                    tsdc."section",
                    tsdc."sectionIndex",
                    tsdc."numQuestions"
                FROM get_test_section_difficulty_counts(original_test_id) tsdc
                WHERE tsdc."difficultyLevel" = 'easy'
                AND tsdc."numQuestions" > 0  -- ✅ Skip sections with 0 questions
                ORDER BY tsdc."sectionIndex"
            ),
            section_start_positions AS (
                -- Compute section start numbers dynamically, ensuring first section starts at 1
                SELECT 
                    "section",
                    COALESCE(
                        SUM("numQuestions") OVER (ORDER BY "sectionIndex" ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) + 1,
                        1
                    ) AS "startNum"
                FROM section_question_counts
            )
            SELECT jsonb_agg(jsonb_build_array("section", "startNum")) 
            FROM section_start_positions
            WHERE "startNum" IS NOT NULL  -- ✅ Skip NULL values
        ),
        "importUrl", "resultMsgHtml", "showAnswer", "year", "exam", "pdfURL", "userId", "scholarship", 
        "reviewAt", "discussionEnd", "seqId", "allowPracticeMode", "chapterId", "lockSection"
    FROM "Test"
    WHERE "id" = original_test_id
    RETURNING "id" INTO easy_test_id;

    -- Repeat for Medium and Difficult difficulty levels
    INSERT INTO "Test" ("name", "description", "instructions", "durationInMin", "positiveMarks", "negativeMarks", 
        "ownerId", "ownerType", "creatorId", "createdAt", "updatedAt", "numQuestions", "syllabus", 
        "free", "sections", "importUrl", "resultMsgHtml", "showAnswer", "year", "exam", "pdfURL", 
        "userId", "scholarship", "reviewAt", "discussionEnd", "seqId", "allowPracticeMode", "chapterId", "lockSection")
    SELECT 
        CONCAT("name", ' - Medium'), "description", "instructions", "durationInMin", "positiveMarks", "negativeMarks", 
        "ownerId", "ownerType", "creatorId", NOW(), NOW(), 0, "syllabus", "free", 
        (
            WITH section_question_counts AS (
                SELECT 
                    tsdc."section",
                    tsdc."sectionIndex",
                    tsdc."numQuestions"
                FROM get_test_section_difficulty_counts(original_test_id) tsdc
                WHERE tsdc."difficultyLevel" = 'medium'
                AND tsdc."numQuestions" > 0
                ORDER BY tsdc."sectionIndex"
            ),
            section_start_positions AS (
                SELECT
