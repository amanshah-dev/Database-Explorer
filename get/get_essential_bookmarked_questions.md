CREATE FUNCTION get_essential_bookmarked_questions(p_userid INTEGER, p_courseid INTEGER DEFAULT NULL::INTEGER, p_topicid INTEGER DEFAULT NULL::INTEGER)
    RETURNS TABLE("questionId" INTEGER, "chapterId" INTEGER, question TEXT, explanation TEXT, "correctOptionIndex" INTEGER, ncert BOOLEAN, "userAnswer" INTEGER)
    LANGUAGE plpgsql
AS
$$
BEGIN
    -- If courseId is NULL, execute the first query
    IF p_courseId IS NULL THEN
        RETURN QUERY
        WITH UserHasActiveCourse AS (
            SELECT EXISTS (
                SELECT 1
                FROM "UserCourse" "UC"
                WHERE "UC"."userId" = p_userId
                  AND "UC"."courseId" IN (3125, 3622, 3653, 4247, 4148)
                  AND "UC"."expiryAt" > NOW()
                  AND "UC"."startedAt" < NOW()
            ) AS "hasAccess"
        )
        SELECT DISTINCT ON ("Q"."id")
            "Q"."id" AS "questionId",
            "Q"."topicId" AS "chapterId",
            "Q"."question",
            CASE
                WHEN (SELECT "hasAccess" FROM UserHasActiveCourse) THEN "Q"."explanation"
                ELSE ''::TEXT
            END AS "explanation",
            "Q"."correctOptionIndex",
            "Q"."ncert"::BOOLEAN AS "ncert",
            COALESCE("A"."userAnswer", NULL) AS "userAnswer"  -- Ensures NULL handling
        FROM "BookmarkQuestion" "BQ"
        JOIN "Question" "Q" ON "Q"."id" = "BQ"."questionId"
        LEFT JOIN "Answer" "A" ON "A"."questionId" = "Q"."id"
                                AND "A"."userId" = p_userId
        WHERE "BQ"."userId" = p_userId
        AND (p_topicId IS NULL OR "Q"."topicId" = p_topicId)
        ORDER BY "Q".id, "Q"."topicId";

    -- If courseId is provided, execute the second query
    ELSE
        RETURN QUERY
        WITH UserHasActiveCourse AS (
            SELECT EXISTS (
                SELECT 1
                FROM "UserCourse" "UC"
                WHERE "UC"."userId" = p_userId
                  AND "UC"."courseId" IN (3125, 3622, 3653, 4247, 4148)
                  AND "UC"."expiryAt" > NOW()
                  AND "UC"."startedAt" < NOW()
            ) AS "hasAccess"
        )
        SELECT DISTINCT ON ("Q"."id")
            "Q"."id" AS "questionId",
            "Q"."topicId" AS "chapterId",
            "Q"."question",
            CASE
                WHEN (SELECT "hasAccess" FROM UserHasActiveCourse) THEN "Q"."explanation"
                ELSE ''::TEXT
            END AS "explanation",
            "Q"."correctOptionIndex",
            "Q"."ncert"::BOOLEAN AS "ncert",
            COALESCE("A"."userAnswer", NULL) AS "userAnswer"  -- Ensures NULL handling
        FROM "BookmarkQuestion" "BQ"
        JOIN "TestQuestion" "TQ" ON "TQ"."questionId" = "BQ"."questionId"
        JOIN "Test" "TE" ON "TE"."id" = "TQ"."testId"
        JOIN "ConsolidatedTest" "CT" ON "CT"."testId" = "TE"."id"
        JOIN "Question" "Q" ON "Q"."id" = "BQ"."questionId"
        LEFT JOIN "Answer" "A" ON "A"."questionId" = "Q"."id"
                                AND "A"."userId" = p_userId
        WHERE
          "CT"."courseId" = p_courseId
          AND "BQ"."userId" = p_userId
          AND (p_topicId IS NULL OR "Q"."topicId" = p_topicId)
        ORDER BY "Q"."id";
    END IF;
END;
$$;

ALTER FUNCTION get_essential_bookmarked_questions(INTEGER, INTEGER, INTEGER) OWNER TO learner;
