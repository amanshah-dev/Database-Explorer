CREATE FUNCTION "MergeOfflineTestAttemptResults"(testid INTEGER, usetestid INTEGER) RETURNS VOID
LANGUAGE plpgsql
AS
$$
BEGIN
  EXECUTE "CalculateOfflineTestAttemptResults"(testId);

  UPDATE "OfflineTestAttempt" 
  SET "userId" = "rawUserId", 
      "testId" = useTestId, 
      "updatedAt" = CURRENT_TIMESTAMP, 
      "centreId" = COALESCE("OfflineTestAttempt"."centreId", "UserCentre"."centreId") 
  FROM "User", "UserCentre" 
  WHERE "User"."id" = "rawUserId" 
    AND "UserCentre"."userId" = "rawUserId" 
    AND "OfflineTestAttempt"."userId" IS NULL 
    AND "OfflineTestAttempt"."result" IS NOT NULL 
    AND "OfflineTestAttempt"."testId" IS NULL 
    AND "OfflineTestAttempt"."rawTestId" = testId;

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
    AND "OfflineTestAttempt"."rawTestId" = testId;

  INSERT INTO "TestAttempt" ("testId", "userId", "userAnswers", "result", "offlineTestAttemptId", "completed")
  SELECT "testId", "userId", "quesAnswers", "result", "id", TRUE 
  FROM "OfflineTestAttempt" 
  WHERE "rawTestId" = testId 
    AND "result" IS NOT NULL 
    AND "quesAnswers" IS NOT NULL 
    AND "testId" IS NOT NULL 
    AND "userId" IS NOT NULL 
  ON CONFLICT ("offlineTestAttemptId") 
  DO UPDATE 
    SET "result" = EXCLUDED."result", 
        "userAnswers" = EXCLUDED."userAnswers";

  UPDATE "OfflineTestAttempt" 
  SET "rank" = t1."rank" 
  FROM (
    SELECT "score", "attemptId", "mode", RANK() OVER (ORDER BY "score" DESC) AS "rank" 
    FROM (
      SELECT ("result"->>'totalMarks')::INTEGER AS "score", 
             "TestAttempt"."id" AS "attemptId", 
             'Online' AS "mode" 
      FROM "TestAttempt", "Test" 
      WHERE "TestAttempt"."testId" = useTestId 
        AND "TestAttempt"."testId" = "Test"."id" 
        AND "TestAttempt"."completed" = TRUE 
        AND "TestAttempt"."finishedAt" < "Test"."reviewAt" 
        AND "result" IS NOT NULL 
        AND "TestAttempt"."userId" NOT IN (
          SELECT "userId" 
          FROM "TestAttempt" ta 
          WHERE ta."testId" = useTestId 
            AND ta."offlineTestAttemptId" IS NOT NULL
        )
      UNION
      SELECT ("result"->>'totalMarks')::INTEGER AS "score", 
             "OfflineTestAttempt"."id" AS "attemptId", 
             'Offline' AS "mode" 
      FROM "OfflineTestAttempt" 
      WHERE (("OfflineTestAttempt"."rawTestId" = testId) 
             OR ("OfflineTestAttempt"."rawTestId" IN (
               SELECT "dupId" 
               FROM "DuplicateTest" 
               WHERE "origId" = (
                 SELECT "origId" 
                 FROM "DuplicateTest" dt 
                 WHERE dt."dupId" = testId
               )
             ))
      ) 
      AND "result" IS NOT NULL
    ) t 
    ORDER BY "score" DESC
  ) t1 
  WHERE t1."attemptId" = "OfflineTestAttempt"."id";

  -- Update any existing online test attempt if any by the user
  UPDATE "TestAttempt" ta 
  SET "completed" = FALSE, 
      "updatedAt" = ta1."createdAt" 
  FROM "OfflineTestAttempt" ota,
       "TestAttempt" ta1
  WHERE ta."completed" = TRUE 
    AND ta1."completed" = TRUE 
    AND ta."testId" = ta1."testId" 
    AND ta."userId" = ta1."userId" 
    AND ta."offlineTestAttemptId" IS NULL 
    AND ta1."offlineTestAttemptId" = ota."id" 
    AND ota."testId" = ta1."testId" 
    AND ota."testId" = useTestId 
    AND ota."rawTestId" = testId 
    AND ota."userId" = ta."userId" 
    AND ota."testId" = ta."testId" 
    AND ota."userId" = ta1."userId" 
    AND ota."testId" = ta1."testId" 
    AND ta1."id" != ta."id";
END
$$;

ALTER FUNCTION "MergeOfflineTestAttemptResults"(INTEGER, INTEGER) OWNER TO learner;
