CREATE FUNCTION "duplicateChapterQuestions"(chapterid INTEGER) RETURNS VOID
    LANGUAGE plpgsql
AS
$$
DECLARE 
  rec RECORD;
BEGIN 
  FOR rec IN 
    SELECT "questionId" 
    FROM "ChapterQuestion"
    WHERE "ChapterQuestion"."chapterId" = chapterId
  LOOP
    PERFORM "duplicateChapterQuestions"(rec."questionId", chapterId);
  END LOOP;
END
$$;

ALTER FUNCTION "duplicateChapterQuestions"(integer) OWNER TO learner;
