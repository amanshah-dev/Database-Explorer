create function "PopulateBackupTempTestQuestions"(batchtestid integer) returns void
    language plpgsql
as
$$
DECLARE
    tempTestId INTEGER;
    backupTempTestId INTEGER;
    testPatternId INTEGER;
    generated_questions RECORD;
BEGIN
    -- Retrieve tempTestId and examPatternId from BatchTest
    SELECT "backupTempTestId", "tempTestId", "testPatternId"
    INTO backupTempTestId, tempTestId, testPatternId
    FROM "BatchTest"
    WHERE "id" = batchTestId;

    -- Delete existing entries for the given batchTestId in TempTestQuestion
    DELETE FROM "TempTestQuestion"
    WHERE "tempTestId" = backupTempTestId;

    -- Generate questions using "GenerateQuestionsFromQuesBank"
    INSERT INTO "TempTestQuestion" ("tempTestId", "chapterId", "section", "questionId", "seqNum")
    SELECT
        backupTempTestId,
        "chapter_id",
        "section_name",
        "question_id",
        row_number() OVER (order by "section_name" ASC, "chapter_id" ASC) as row_num
    FROM "GenerateQuestionsFromQuesBank"(
        testPatternId,
        "CreateSectionChapterNumQuestionArrForBatchTest"(batchTestId),
        ('{"function_name": "remove_selected_temp_questions", "args": {"tempTestId": ' || tempTestId::TEXT || '}}')::jsonb
    );

END;
$$;

alter function "PopulateBackupTempTestQuestions"(integer) owner to neetprep_rw;
