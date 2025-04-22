CREATE FUNCTION "GetUserScoreBoosterDetail"(p_userid bigint, p_scheduleid bigint)
    RETURNS TABLE(
        "userId" bigint,
        "scheduleId" bigint,
        "scheduleName" text,
        "scheduleItemId" bigint,
        "scheduleItemName" text,
        "scheduledAt" timestamp with time zone,
        "topicId" bigint,
        "topicName" text,
        "subjectId" bigint,
        "subjectName" text,
        questions bigint,
        "correctQuestions" bigint,
        "assetId" bigint,
        "assetName" text,
        "assetLink" text,
        "platformQuestions" bigint,
        "platformCorrectQuestions" bigint
    )
    STABLE
    LANGUAGE sql
AS
$$
WITH 
    -- Get user and schedule information first
    user_schedule AS (
        SELECT 
            "US"."userId",
            "US"."externalScheduleId" AS "scheduleId",
            "S".name AS "scheduleName"
        FROM "UserSchedule" "US"
        JOIN "Schedule" "S" ON "US"."externalScheduleId" = "S".id
        WHERE "US"."userId" = p_userId
        AND "US"."externalScheduleId" = p_scheduleId
        LIMIT 1 -- We only need one row since userId and scheduleId uniquely identify the record
    ),
    
    -- Get all schedule items for this schedule
    schedule_items AS (
        SELECT 
            "SI".id AS "scheduleItemId",
            "SI".name AS "scheduleItemName",
            "SI"."scheduledAt"
        FROM "ScheduleItem" "SI"
        WHERE "SI"."scheduleId" = p_scheduleId
    ),
    
    -- Get relevant subjects
    subjects AS (
        SELECT 
            "SUB".id AS "subjectId",
            "SUB".name AS "subjectName"
        FROM "Subject" "SUB"
        WHERE "SUB".id = ANY (ARRAY [53, 54, 55, 56])
    ),
    
    -- Get relevant topics that belong to our subjects
    topics AS (
        SELECT
            "T".id AS "topicId",
            "T".name AS "topicName",
            "T"."subjectId"
        FROM "Topic" "T"
        JOIN subjects "SUB" ON "T"."subjectId" = "SUB"."subjectId"
    ),
    
    -- Get schedule item chapters - filtering by scheduleId early to reduce data volume
    schedule_item_chapters AS (
        SELECT 
            "SIC".id AS "scheduleItemChapterId",
            "SIC"."scheduleItemId",
            "SIC"."topicId",
            "T"."topicName",
            "T"."subjectId",
            "SUB"."subjectName"
        FROM "ScheduleItemChapter" "SIC"
        JOIN topics "T" ON "SIC"."topicId" = "T"."topicId"
        JOIN subjects "SUB" ON "T"."subjectId" = "SUB"."subjectId"
        WHERE EXISTS (
            SELECT 1 FROM "ScheduleItem" "SI" 
            WHERE "SI"."id" = "SIC"."scheduleItemId" 
            AND "SI"."scheduleId" = p_scheduleId
        )
    ),
    
    -- Get assets with filtering to reduce data volume
    schedule_item_assets AS (
        SELECT 
            "SIA".id AS "assetId",
            "SIA"."assetName",
            "SIA"."assetLink",
            "SIC"."topicId",
            "SIC"."scheduleItemId"
        FROM schedule_item_chapters "SIC"
        JOIN "ScheduleItemAsset" "SIA" ON "SIA"."scheduleItemChapterId" = "SIC"."scheduleItemChapterId"
    ),
    
    -- Get user analytics with direct filtering
    user_reported_analytics AS (
        SELECT DISTINCT ON ("topicId") 
            "topicId",
            questions,
            "correctQuestions"
        FROM "UserReportedAnalytic"
        WHERE "userId" = p_userId
        ORDER BY "topicId", "scheduleItemId" DESC
    ),
    
    -- Pre-calculate platform analytics with direct filtering by userId
    platform_analytics AS (
        SELECT 
            "Q"."topicId",
            COUNT("A".id) AS "answerCount",
            COUNT(CASE WHEN "A"."userAnswer" = "Q"."correctOptionIndex" THEN 1 END) AS "correctAnswerCount"
        FROM "Answer" "A"
        JOIN "Question" "Q" ON "A"."questionId" = "Q".id
        WHERE "A"."userId" = p_userId
        GROUP BY "Q"."topicId"
    )

-- Combine all the data efficiently
SELECT 
    p_userId AS "userId",
    p_scheduleId AS "scheduleId",
    "US"."scheduleName",
    "SI"."scheduleItemId",
    "SI"."scheduleItemName",
    "SI"."scheduledAt",
    "SIC"."topicId",
    "SIC"."topicName",
    "SIC"."subjectId",
    "SIC"."subjectName",
    "URA".questions,
    "URA"."correctQuestions",
    "SIA"."assetId",
    "SIA"."assetName",
    "SIA"."assetLink",
    COALESCE("PA"."answerCount", 0::bigint) AS "platformQuestions",
    COALESCE("PA"."correctAnswerCount", 0::bigint) AS "platformCorrectQuestions"
FROM user_schedule "US"
CROSS JOIN schedule_items "SI" -- Using CROSS JOIN since we already filtered schedule_items by scheduleId
LEFT JOIN schedule_item_chapters "SIC" ON "SI"."scheduleItemId" = "SIC"."scheduleItemId"
LEFT JOIN schedule_item_assets "SIA" ON "SI"."scheduleItemId" = "SIA"."scheduleItemId" AND "SIC"."topicId" = "SIA"."topicId"
LEFT JOIN user_reported_analytics "URA" ON "SIC"."topicId" = "URA"."topicId"
LEFT JOIN platform_analytics "PA" ON "SIC"."topicId" = "PA"."topicId";
$$;

ALTER FUNCTION "GetUserScoreBoosterDetail"(bigint, bigint) OWNER TO learner;
