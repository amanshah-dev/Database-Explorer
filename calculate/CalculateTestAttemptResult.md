-- Create function to calculate and update the test attempt result based on testid, userid, and testattemptid
create function "CalculateTestAttemptResult"(testid integer, userid integer, testattemptid integer) returns void
    language plpgsql
as
$$
DECLARE
  test_result JSONB;
BEGIN
  -- Get the test result using another function
  SELECT "GetTestAttemptResult"(testattemptid) INTO test_result;

  -- Update result of the testattempt only if there is result to update
  IF test_result IS NOT NULL THEN
    -- Update the current test attempt with the result and mark it as completed
    UPDATE "TestAttempt"
      SET "completed" = TRUE,
        "finishedAt" = CURRENT_TIMESTAMP,
        "updatedAt" = CURRENT_TIMESTAMP,
        "result" = test_result
    WHERE "TestAttempt"."userId" = userid AND
      "TestAttempt"."testId" = testid AND
      "TestAttempt"."completed" IS FALSE AND
      "TestAttempt"."id" = testattemptid;

    -- Update other test attempts by the user where completed is true to false
    UPDATE "TestAttempt"
      SET "completed" = FALSE,
        "updatedAt" = CURRENT_TIMESTAMP
    FROM "TestAttempt" tac
    WHERE "TestAttempt"."userId" = userid AND
      "TestAttempt"."testId" = testid AND
      "TestAttempt"."id" != testattemptid AND
      "TestAttempt"."completed" = TRUE AND
      tac."id" = testattemptid AND
      tac."completed" = TRUE;
  END IF;
END
$$;

-- Alter function owner
alter function "CalculateTestAttemptResult"(integer, integer, integer) owner to neetprep_rw;
