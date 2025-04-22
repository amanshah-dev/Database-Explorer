-- Create function to calculate the user's accuracy by topic
create function calculate_user_topic_accuracy(p_userid integer, p_limit integer DEFAULT 100) returns void
    language plpgsql
as
$$
BEGIN
    WITH LevelBoundaries AS (
        /* Generate levels in increments of 10 from 0 to 100. */
        SELECT 0 AS "minLevel", 10 AS "maxLevel"
        UNION ALL SELECT 10, 20
        UNION ALL SELECT 20, 30
        UNION ALL SELECT 30, 40
        UNION ALL SELECT 40, 50
        UNION ALL SELECT 50, 60
        UNION ALL SELECT 60, 70
        UNION ALL SELECT 70, 80
        UNION ALL SELECT 80, 90
        UNION ALL SELECT 90, 100
    ),
    FilteredAnswers AS (
        /* If p_limit is provided, limit to the most recent p_limit answers per topic. */
        SELECT *
        FROM (
            SELECT 
                "A".*, 
                ROW_NUMBER() OVER (PARTITION BY "Q"."topicId" ORDER BY "A"."createdAt" DESC) AS "rowNumber"
            FROM "Answer" AS "A"
            INNER JOIN "Question" AS "Q"
                ON "A"."questionId" = "Q"."id"
            WHERE "A"."userId" = p_userId
        ) AS rankedAnswers
        WHERE p_limit IS NULL OR "rowNumber" <= p_limit
    ),
    UserTopicAnswers AS (
        /* Join FilteredAnswers with QuestionAnalytics and group by user, topic, and level ranges. */
        SELECT
            "A"."userId",
            "Q"."topicId",
            "Q"."subjectId",
            L."minLevel" AS "minLevel",
            L."maxLevel" AS "maxLevel",
            COUNT(DISTINCT("Q"."id")) AS "answerCount",
            COUNT(DISTINCT(CASE WHEN "A"."userAnswer" = "Q"."correctOptionIndex" THEN "Q"."id" ELSE NULL END)) AS "correctAnswerCount"
        FROM FilteredAnswers AS "A"
        INNER JOIN "QuestionAnalytics" AS "QA"
            ON "A"."questionId"
