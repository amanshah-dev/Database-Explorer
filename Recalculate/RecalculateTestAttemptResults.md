create function "RecalculateTestAttemptResults"(testid integer) returns void
    language plpgsql
as
$$
DECLARE
  testattemptid integer;
  userid integer;
  test_result JSONB;
BEGIN
  /* Loop through all completed test attempts for the given test and recompute & update their results */
  FOR testattemptid, userid IN
    SELECT "id", "userId"
    FROM "TestAttempt"
    WHERE "testId" = testid
    AND "completed" = TRUE
  LOOP
    SELECT "GetTestAttemptResult"(testattemptid) INTO test_result;
    UPDATE "TestAttempt"
      SET "result" = test_result,
      "updatedAt" = NOW()
    WHERE
      "id" = testattemptid AND
      "result" IS NOT NULL AND
      "completed" = TRUE;
  END LOOP;
END
$$;

alter function "RecalculateTestAttemptResults"(integer) owner to neetprep_rw;
