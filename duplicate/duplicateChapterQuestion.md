CREATE FUNCTION "duplicateChapterQuestions"(questionid INTEGER, chapterid INTEGER) RETURNS VOID
    LANGUAGE plpgsql
AS
$$
BEGIN
  WITH asQuestion AS (
    INSERT INTO "Question" (
        "question", "explanation", "options", "correctOptionIndex", 
        "type", "deleted", "paidAccess", "level", "sequenceId", 
        "orignalQuestionId", "topicId", "subjectId", "createdAt", "updatedAt"
    )
    SELECT "question", "explanation", "options", "correctOptionIndex", 
           "type", "deleted", "paidAccess", "level", "sequenceId", 
           "id", "topicId", "subjectId", current_timestamp, current_timestamp
    FROM "Question" 
    WHERE "id" = questionId
    RETURNING id
  )
  UPDATE "ChapterQuestion"
  SET "questionId" = (SELECT id FROM asQuestion)
  WHERE "chapterId" = chapterid AND "questionId" = questionid;
END
$$;

ALTER FUNCTION "duplicateChapterQuestions"(integer, integer) OWNER TO learner;
