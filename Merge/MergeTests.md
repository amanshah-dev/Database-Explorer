CREATE FUNCTION "MergeTests"(intotestid INTEGER, fromtestid INTEGER) RETURNS VOID
LANGUAGE plpgsql
AS
$$
DECLARE
  count INTEGER;
BEGIN
  SELECT count(*) 
  FROM "TestQuestion" 
  WHERE "testId" = intoTestId 
  INTO count;

  INSERT INTO "TestQuestion" ("testId", "questionId", "seqNum") 
  SELECT 
    intoTestId, 
    tq."questionId", 
    (count + row_number() OVER (ORDER BY tq."seqNum" ASC, tq."questionId" ASC)) 
  FROM "TestQuestion" tq, "Question" q
  WHERE tq."testId" = fromTestId 
    AND q."id" = tq."questionId" 
    AND q."deleted" = FALSE 
  ON CONFLICT ("testId", "questionId") DO NOTHING;

  UPDATE "Test" 
  SET "numQuestions" = (
    SELECT count(*) 
    FROM "TestQuestion", "Question" 
    WHERE "testId" = intoTestId 
      AND "Question"."id" = "TestQuestion"."questionId" 
      AND "Question"."deleted" = FALSE
  ) 
  WHERE "id" = intoTestId;
END
$$;

ALTER FUNCTION "MergeTests"(INTEGER, INTEGER) OWNER TO learner;
