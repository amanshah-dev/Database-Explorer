CREATE FUNCTION get_bookmarked_questions_by_course(course_id INTEGER)
    RETURNS TABLE(chapterid INTEGER, subjectid INTEGER, topicname TEXT, subjectname TEXT, bookmarkedquestionscount INTEGER)
    LANGUAGE plpgsql
AS
$$
BEGIN
    -- Check if the course_id is valid
    IF course_id NOT IN (3125, 3653, 3622, 4775, 4247, 3488, 4577) THEN
        RAISE EXCEPTION 'Invalid course_id: %', course_id;
    END IF;

    -- Return the query based on the course_id
    RETURN QUERY
    SELECT 
        "Topic"."id" AS "chapterId", 
        "Topic"."subjectId",
        "Topic"."name" AS "topicName",
        "Subject"."name" AS "subjectName",
        COUNT(DISTINCT "BookmarkQuestion"."questionId") AS "bookmarkedQuestionsCount"
    FROM 
        "BookmarkQuestion"
    JOIN 
        "TestQuestion" ON "TestQuestion"."questionId" = "BookmarkQuestion"."questionId"
    JOIN 
        "Topic" ON "Topic"."id" = "TestQuestion"."topicId"
    JOIN 
        "Subject" ON "Subject"."id" = "Topic"."subjectId"
    
    -- Conditional joins based on the course_id
    LEFT JOIN 
        "CourseChapter" 
        ON "CourseChapter"."chapterId" = "Topic"."id"
        AND "CourseChapter"."courseId" = course_id
    LEFT JOIN 
        "CourseTest" 
        ON "CourseTest"."testId" = "TestQuestion"."testId"
        AND "CourseTest"."courseId" = course_id

    WHERE 
        "BookmarkQuestion"."userId" = 2035230  -- Specific user
        AND (
            -- Handling course-specific logic
            (course_id IN (3125, 3653, 3622, 4775, 4247) AND "CourseChapter"."courseId" IS NOT NULL)
            OR (course_id = 3488 AND "CourseChapter"."courseId" IS NOT NULL)
            OR (course_id = 4577 AND "CourseTest"."courseId" IS NOT NULL)
        )
    GROUP BY 
        "Topic"."id", "Topic"."subjectId", "Topic"."name", "Subject"."name"
    ORDER BY 
        "bookmarkedQuestionsCount" DESC;

END;
$$;

ALTER FUNCTION get_bookmarked_questions_by_course(INTEGER) OWNER TO learner;
