CREATE FUNCTION get_bookmarked_questions(userid INTEGER, courseid INTEGER DEFAULT NULL::INTEGER)
    RETURNS TABLE("chapterId" INTEGER, "subjectId" INTEGER, "topicName" TEXT, "subjectName" TEXT, count INTEGER)
    LANGUAGE plpgsql
AS
$$
BEGIN
    IF courseId IS NULL THEN
        -- Case when "courseId" is not provided (default subjects: 53, 54, 55, 56)
        RETURN QUERY
        SELECT 
            "q"."topicId" AS "chapterId",
            "q"."subjectId",
            (SELECT "t"."name"::TEXT FROM "Topic" "t" WHERE "t"."id" = "q"."topicId") AS "topicName",
            (SELECT "s"."name"::TEXT FROM "Subject" "s" WHERE "s"."id" = "q"."subjectId") AS "subjectName",
            COUNT(*)::INT AS "count"  -- Rename column to "count"
        FROM "BookmarkQuestion" "bq"
        JOIN "Question" "q" ON "q"."id" = "bq"."questionId"
        WHERE "bq"."userId" = userId
          AND "q"."subjectId" = ANY (ARRAY[53, 54, 55, 56]) -- Filter for default subjects
        GROUP BY "q"."topicId", "q"."subjectId"
        ORDER BY "q"."subjectId";

    ELSE
        -- Case when "courseId" is provided (uses "ConsolidatedTest")
        RETURN QUERY
        SELECT 
            "q"."topicId" AS "chapterId",
            "q"."subjectId",
            "t"."name"::TEXT AS "topicName",  -- Cast "VARCHAR" to "TEXT"
            "s"."name"::TEXT AS "subjectName",  -- Cast "VARCHAR" to "TEXT"
            COUNT("bq"."questionId")::INT AS "count"  -- Rename column to "count"
        FROM "ConsolidatedTest" "ct"
        JOIN "Test" "te" ON "te"."id" = "ct"."testId"
        JOIN "TestQuestion" "tq" ON "tq"."testId" = "te"."id"
        JOIN "Question" "q" ON "q"."id" = "tq"."questionId"
        JOIN "Topic" "t" ON "t"."id" = "q"."topicId"
        JOIN "Subject" "s" ON "s"."id" = "q"."subjectId"
        JOIN "BookmarkQuestion" "bq" 
            ON "bq"."questionId" = "tq"."questionId" 
            AND "bq"."userId" = userId
        WHERE "ct"."courseId" = courseId  -- Uses the given "courseId"
        GROUP BY "q"."topicId", "q"."subjectId", "t"."name", "s"."name"
        ORDER BY "q"."subjectId";
    END IF;
END;
$$;

ALTER FUNCTION get_bookmarked_questions(INTEGER, INTEGER) OWNER TO learner;
