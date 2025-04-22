CREATE FUNCTION "MergeOfflineTestAttemptResults"(testid INTEGER, usetestid INTEGER, offlinetestattemptid INTEGER) RETURNS VOID
LANGUAGE plpgsql
AS
$$
BEGIN
  EXECUTE "CalculateOfflineTestAttemptResults"(testId, offlineTestAttemptId);

  UPDATE "OfflineTestAttempt" 
  SET "userId" = "rawUserId", 
      "testId" = useTestId, 
      "updatedAt" = CURRENT_TIMESTAMP, 
      "centreId" = COALESCE("OfflineTestAttempt"."centreId", "UserCentre"."centreId") 
  FROM "User", "UserCentre" 
  WHERE "User"."id" = "rawUserId" 
    AND "UserCentre"."userId" = "rawUserId" 
    AND "OfflineTestAttempt"."userId" IS NULL 
    AND "OfflineTestAttempt"."testId" IS NULL 
    AND "OfflineTestAttempt"."result" IS NOT NULL 
    AND "OfflineTestAttempt"."id" = offlineTestAttemptId;

  -- Update wrong user id if needed
  UPDATE "OfflineTestAttempt" 
  SET "userId" = "rawUserId", 
      "testId" = useTestId, 
      "updatedAt" = CURRENT_TIMESTAMP, 
      "centreId" = COALESCE("OfflineTestAttempt"."centreId", "UserCentre"."centreId") 
  FROM "User", "UserCentre" 
  WHERE "User"."id" = "rawUserId" 
    AND "UserCentre"."userId" = "rawUserId" 
    AND "OfflineTestAttempt"."userId" != "OfflineTestAttempt"."rawUserId" 
    AND "OfflineTestAttempt"."result" IS NOT NULL 
    AND "OfflineTestAttempt"."id" = offlineTestAttemptId;

  -- Use offline raw user id mapping for updating user result now
  UPDATE "OfflineTestAttempt" 
  SET "userId" = "UserOfflineRawUser"."userId", 
      "testId" = useTestId, 
      "updatedAt" = CURRENT_TIMESTAMP 
  FROM "UserOfflineRawUser" 
  WHERE "UserOfflineRawUser"."rawUserId" = "OfflineTestAttempt"."rawUserId" 
    AND "OfflineTestAttempt"."userId" IS NULL 
    AND "OfflineTestAttempt"."testId" IS NULL 
    AND "OfflineTestAttempt"."result" IS NOT NULL 
    AND "OfflineTestAttempt"."id" = offlineTestAttemptId;

  -- Update wrong user id if needed
  UPDATE "OfflineTestAttempt" 
  SET "userId" = "UserOfflineRawUser"."userId", 
      "testId" = useTestId, 
      "updatedAt" = CURRENT_TIMESTAMP 
  FROM "UserOfflineRawUser" 
  WHERE "UserOfflineRawUser"."rawUserId" = "OfflineTestAttempt"."rawUserId" 
    AND "OfflineTestAttempt"."userId" != "UserOfflineRawUser"."userId" 
    AND "OfflineTestAttempt"."result" IS NOT NULL 
    AND "OfflineTestAttempt"."id" = offlineTestAttemptId;

  -- Update wrong test id if needed
  UPDATE "OfflineTestAttempt" 
  SET "testId" = useTestId 
  FROM "DuplicateTest" 
  WHERE "OfflineTestAttempt"."id" = offlineTestAttemptId 
    AND "OfflineTestAttempt"."testId" != useTestId 
    AND "DuplicateTest"."dupId" = "OfflineTestAttempt"."rawTestId" 
    AND "DuplicateTest"."origId" = useTestId;

  INSERT INTO "TestAttempt" ("testId", "userId", "userAnswers", "result", "offlineTestAttemptId", "completed")
  SELECT "testId", "userId", "quesAnswers", "result", "id", TRUE 
  FROM "OfflineTestAttempt" 
  WHERE "rawTestId" = testId 
    AND "result" IS NOT NULL 
    AND "quesAnswers" IS NOT NULL 
    AND "testId" IS NOT NULL 
    AND "userId" IS NOT NULL 
    AND "OfflineTestAttempt"."id" = offlineTestAttemptId 
  ON CONFLICT ("offlineTestAttemptId") 
  DO UPDATE 
    SET "result" = EXCLUDED."result", 
        "userAnswers" = EXCLUDED."userAnswers", 
        "userId" = EXCLUDED."userId", 
        "testId" = EXCLUDED."testId";

  -- Update any existing online test attempt if any by the user
  UPDATE "TestAttempt" ta 
  SET "completed" = FALSE, 
      "updatedAt" = ta1."createdAt" -- this data is just to help resolve any issue that may happen because of this change
  FROM "TestAttempt" ta1
  WHERE ta."completed" = TRUE 
    AND ta1."completed" = TRUE 
    AND ta."testId" = ta1."testId" 
    AND ta."userId" = ta1."userId" 
    AND ta."offlineTestAttemptId" IS NULL 
    AND ta1."offlineTestAttemptId" = offlineTestAttemptId 
    AND ta1."id" != ta."id";
END
$$;

ALTER FUNCTION "MergeOfflineTestAttemptResults"(INTEGER, INTEGER, INTEGER) OWNER TO learner;
