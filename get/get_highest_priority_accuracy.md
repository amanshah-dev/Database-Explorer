CREATE FUNCTION get_highest_priority_accuracy(
    "inputUserId" INTEGER, 
    "inputSubTopicIds" INTEGER[], 
    "thresholdAccuracyPercentage" INTEGER DEFAULT 80
) 
RETURNS TABLE(
    "userId" INTEGER, 
    "topicId" INTEGER, 
    "subjectId" INTEGER, 
    "minLevel" INTEGER, 
    "maxLevel" INTEGER, 
    "accuracyPercentage" DOUBLE PRECISION, 
    "resultType" TEXT, 
    "correctAnswerCount" INTEGER, 
    "answerCount" INTEGER, 
    "createdAt" TIMESTAMP WITH TIME ZONE, 
    "updatedAt" TIMESTAMP WITH TIME ZONE
) 
LANGUAGE plpgsql
AS
$$
BEGIN
    --------------------------------------------------------------------------------
    -- 1) Return Topic-level accuracy if found
    --------------------------------------------------------------------------------
    RETURN QUERY
    WITH "RelevantTopics" AS (
        SELECT DISTINCT st."topicId"
        FROM "SubTopic" st
        WHERE st."id" = ANY ("inputSubTopicIds")
    )
    SELECT
        uta."userId",
        uta."topicId",
        NULL::INTEGER      AS "subjectId",
        uta."minLevel",
        uta."maxLevel",
        uta."accuracyPercentage",
        'Topic'            AS "resultType",
        uta."correctAnswerCount",
        uta."answerCount",
        uta."createdAt",
        uta."updatedAt"
    FROM "UserTopicAccuracyByLevel" uta
    WHERE uta."userId" = "inputUserId"
      -- Ignore rows where accuracyPercentage > 80
      AND (uta."accuracyPercentage" < "thresholdAccuracyPercentage")
      AND uta."topicId" IN (SELECT rt."topicId" FROM "RelevantTopics" rt)
    ORDER BY uta."maxLevel" DESC
    LIMIT 1;

    -- If the above query returned a row, exit immediately
    IF FOUND THEN
        RETURN;
    END IF;

    --------------------------------------------------------------------------------
    -- 2) If no Topic accuracy, try Subject-level accuracy
    --------------------------------------------------------------------------------
    RETURN QUERY
    WITH "RelevantTopics" AS (
        SELECT DISTINCT st."topicId"
        FROM "SubTopic" st
        WHERE st."id" = ANY ("inputSubTopicIds")
    )
    SELECT
        usa."userId",
        NULL::INTEGER      AS "topicId",
        usa."subjectId",
        usa."minLevel",
        usa."maxLevel",
        usa."accuracyPercentage",
        'Subject'          AS "resultType",
        usa."correctAnswerCount",
        usa."answerCount",
        usa."createdAt",
        usa."updatedAt"
    FROM "UserSubjectAccuracyByLevel" usa
    WHERE usa."userId" = "inputUserId"
      -- Ignore rows where accuracyPercentage > 80
      AND (usa."accuracyPercentage" < "thresholdAccuracyPercentage")
      AND usa."subjectId" IN (
          SELECT DISTINCT uta."subjectId"
          FROM "UserTopicAccuracyByLevel" uta
          WHERE uta."topicId" IN (
              SELECT rt."topicId" FROM "RelevantTopics" rt
          )
      )
    ORDER BY usa."maxLevel" DESC
    LIMIT 1;

    -- If the above query returned a row, exit immediately
    IF FOUND THEN
        RETURN;
    END IF;

    --------------------------------------------------------------------------------
    -- 3) If still no row, return Overall accuracy
    --------------------------------------------------------------------------------
    RETURN QUERY
    SELECT
        ua."userId",
        NULL::INTEGER      AS "topicId",
        NULL::INTEGER      AS "subjectId",
        ua."minLevel",
        ua."maxLevel",
        ua."accuracyPercentage",
        'Overall'          AS "resultType",
        NULL::INTEGER      AS "correctAnswerCount",
        NULL::INTEGER      AS "answerCount",
        ua."createdAt",
        ua."updatedAt"
    FROM "UserAccuracyByLevel" ua
    WHERE ua."userId" = "inputUserId"
      -- Ignore rows where accuracyPercentage > 80
      AND (ua."accuracyPercentage" < "thresholdAccuracyPercentage")
    ORDER BY ua."maxLevel" DESC
    LIMIT 1;

    --------------------------------------------------------------------------------
    -- 4) If absolutely nothing found, return a fallback / “default” row
    --------------------------------------------------------------------------------
    RETURN QUERY
    SELECT
        "inputUserId"                          AS "userId",
        NULL::INTEGER                          AS "topicId",
        NULL::INTEGER                          AS "subjectId",
        70                                     AS "minLevel",
        90                                     AS "maxLevel",
        0.0                                    AS "accuracyPercentage",
        'Default'                              AS "resultType",
        0                                      AS "correctAnswerCount",
        0                                      AS "answerCount",
        now()                                  AS "createdAt",
        now()                                  AS "updatedAt";

    RETURN;
END;
$$;

ALTER FUNCTION get_highest_priority_accuracy(INTEGER, INTEGER[], INTEGER) OWNER TO learner;
