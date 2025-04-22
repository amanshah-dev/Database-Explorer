CREATE FUNCTION "newDuplicateChapter"("oldDuplicateChapter" INTEGER, "newOriginalChapter" INTEGER) RETURNS INTEGER
LANGUAGE plpgsql
AS
$$
DECLARE 
  "returnId" INTEGER; 
BEGIN
  SELECT "NewDuplicateChapter"."dupId" 
  INTO "returnId"
  FROM "DuplicateChapter", 
       "DuplicateChapter" AS "NewDuplicateChapter", 
       "Topic", 
       "Topic" AS "NewTopic", 
       "Subject", 
       "Subject" AS "NewSubject"
  WHERE "Topic"."subjectId" = "Subject"."id"
    AND "NewTopic"."subjectId" = "NewSubject"."id"
    AND "Subject"."courseId" = "NewSubject"."courseId"
    AND "NewDuplicateChapter"."origId" = "newOriginalChapter"
    AND "DuplicateChapter"."dupId" = "oldDuplicateChapter"
    AND "DuplicateChapter"."dupId" = "Topic"."id"
    AND "NewDuplicateChapter"."dupId" = "NewTopic"."id";

  RETURN "returnId";
END
$$;

ALTER FUNCTION "newDuplicateChapter"(INTEGER, INTEGER) OWNER TO learner;
