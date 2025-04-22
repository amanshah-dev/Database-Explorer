create function "PopulateTempTestQuestions"(batchtestid integer) returns void
    language plpgsql
as
$$
DECLARE
    tempTestId INTEGER;
    testPatternId INTEGER;
    generated_questions RECORD;
BEGIN
    
    SELECT "tempTestId", "testPatternId"
    INTO tempTestId, testPatternId
    FROM "BatchTest"
    WHERE "id" = batchTestId;

    -- Delete existing entries for the given tempTestId
    DELETE FROM "TempTestQuestion"
    WHERE "tempTestId" = tempTestId;

    -- Insert new questions into TempTestQuestion table
    INSERT INTO "TempTestQuestion" ("tempTestId", "chapterId", "section", "questionId", "seqNum")
    SELECT tempTestId, "chapter_id", "section_name", "question_id", 
           row_number() OVER (order by "section_name" ASC, "chapter_id" ASC) as row_num
    FROM "GenerateQuestionsFromQuesBank"(
        testPatternId,
        "CreateSectionChapterNumQuestionArrForBatchTest"(batchTestId),
        ('{"function_name": "remove_selected_temp_questions", "args": {"tempTestId": ' || tempTestId::TEXT || '}}')::jsonb
    );

END;
$$;

alter function "PopulateTempTestQuestions"(integer) owner to neetprep_rw;
