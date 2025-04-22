CREATE FUNCTION "InsertSelectQuestions"(chapterid INTEGER) RETURNS VOID
LANGUAGE plpgsql
AS
$$
BEGIN
  INSERT INTO "ChapterQuestion" ("chapterId", "questionId")
  SELECT 
    (SELECT "dupId" 
     FROM "DuplicateChapter", "SubjectChapter"
     WHERE "origId" = chapterId 
       AND "SubjectChapter"."chapterId" = "DuplicateChapter"."dupId" 
       AND "SubjectChapter"."subjectId" = 727), 
    "id"
  FROM "Question"
  WHERE "id" IN (
    SELECT "questionId" 
    FROM "ChapterQuestion" 
    WHERE "chapterId" = chapterid
  ) 
  AND "deleted" = FALSE 
  AND "explanation" ILIKE '%page%';
END
$$;

ALTER FUNCTION "InsertSelectQuestions"(INTEGER) OWNER TO learner;
