CREATE FUNCTION "InsertRelatedQuestions"(testid INTEGER) RETURNS VOID
LANGUAGE plpgsql
AS
$$
DECLARE
  rec RECORD;
BEGIN
  FOR rec IN
    SELECT "questionId" FROM "TestQuestion" WHERE "testId" = testId
  LOOP
    PERFORM "InsertRelatedQuestions"(rec."questionId", testId);
  END LOOP;
END
$$;

ALTER FUNCTION "InsertRelatedQuestions"(INTEGER) OWNER TO learner;
