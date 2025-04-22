create function "RecalculateTestAttemptResult"(testid integer, userid integer, testattemptid integer) returns void
    language plpgsql
as
$$
DECLARE
  test_result JSONB;
BEGIN
  SELECT "GetTestAttemptResult"(testattemptid) INTO test_result;
  
  UPDATE "TestAttempt"
  SET "result" = test_result,
      "updatedAt" = NOW()
  WHERE "id" = testattemptid
    AND "testId" = testId
    AND "userId" = userId
    AND "completed" = TRUE
    AND "result" IS NOT NULL;
END
$$;

alter function "RecalculateTestAttemptResult"(integer, integer, integer) owner to learner;
