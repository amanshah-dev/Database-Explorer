
create function "TestAttemptDetailFunc"(usertestattemptid integer) returns boolean
    language plpgsql
as
$$
DECLARE
    test RECORD;
    testId INTEGER := 0;
    showAnswer BOOLEAN := FALSE;
BEGIN
    -- Get the test ID associated with the given user test attempt
    SELECT "testId"
    INTO testId
    FROM "TestAttempt"
    WHERE "id" = usertestattemptid;

    -- Get the details of the test
    SELECT "id", "free", "userId", "showAnswer", "reviewAt"
    INTO test
    FROM "Test"
    WHERE "id" = testId;

    -- If the test's "showAnswer" is false, return false
    IF test."showAnswer" = FALSE THEN
        RETURN FALSE;
    END IF;

    -- If "reviewAt" is set and the current time is before "reviewAt", return false
    IF test."reviewAt" IS NOT NULL AND test."reviewAt" > CURRENT_TIMESTAMP THEN
        RETURN FALSE;
    END IF;

    -- If the test is free and no user is associated, return true
    IF test."free" = TRUE AND test."userId" IS NULL THEN
        RETURN TRUE;
    END IF;

    -- If a user is associated with the test
    IF test."userId" IS NOT NULL THEN
        -- Check if the user has access to the test based on "TARGET_BASED_COURSE_IDS" or "OTHER_COURSE_IDS"
        SELECT COUNT("UserCourse"."id") > 0
        INTO showAnswer
        FROM "UserCourse"
        WHERE "UserCourse"."userId" = test."userId"
          AND "UserCourse"."expiryAt" > NOW()
          AND "UserCourse"."courseId" IN (
              SELECT CAST(jsonb_array_elements("value") AS INTEGER)
              FROM "Constant"
              WHERE "key" IN ('TARGET_BASED_COURSE_IDS', 'OTHER_COURSE_IDS')
          );

        -- If the above check fails, check for course-specific DPP access
        IF showAnswer = FALSE THEN
            SELECT COUNT("UserCourse"."id") > 0
            INTO showAnswer
            FROM "UserCourse"
            WHERE "UserCourse"."userId" = test."userId"
              AND "UserCourse"."expiryAt" > NOW()
              AND "UserCourse"."courseId" IN (
                  SELECT "courseId"
                  FROM "UserDpp"
                  WHERE "UserDpp"."testId" = test."id"
              );
        END IF;

        RETURN showAnswer;
    END IF;

    -- If test is not free, not linked to any course or chapter test, check access for target or other courses
    IF test."free" = FALSE 
       AND test."userId" IS NULL
       AND (SELECT COUNT("CourseTest"."id") FROM "CourseTest" WHERE "testId" = test."id") = 0
       AND (SELECT COUNT("ChapterTest"."id") FROM "ChapterTest" WHERE "testId" = test."id") = 0 THEN
        SELECT COUNT("UserCourse".id) > 0
        INTO showAnswer
        FROM "UserCourse", "TestAttempt"
        WHERE "TestAttempt"."id" = usertestattemptid
          AND "UserCourse"."userId" = "TestAttempt"."userId"
          AND "UserCourse"."expiryAt" > NOW()
          AND "UserCourse"."courseId" IN (
              SELECT CAST(jsonb_array_elements("value") AS INTEGER)
              FROM "Constant"
              WHERE "key" IN ('TARGET_BASED_COURSE_IDS', 'OTHER_COURSE_IDS')
          );

        RETURN showAnswer;
    END IF;

    -- Check if the test is linked to any course test
    IF (SELECT COUNT("CourseTest"."id") FROM "CourseTest" WHERE "testId" = test."id") > 0 THEN
        SELECT COUNT("UserCourse".id) > 0
        INTO showAnswer
        FROM "TestAttempt"
        JOIN "Test" ON "Test".id = "TestAttempt"."testId"
                   AND "TestAttempt"."id" = usertestattemptid
                   AND "TestAttempt"."completed" = TRUE
        LEFT JOIN "CourseTest" ON "CourseTest"."testId" = "Test".id
                              AND "Test".free = FALSE
        LEFT JOIN "UserCourse" ON "UserCourse"."userId" = "TestAttempt"."userId"
                              AND "UserCourse"."expiryAt" > NOW()
                              AND "UserCourse"."courseId" = "CourseTest"."courseId";

        RETURN showAnswer;
    END IF;

    -- Check if the test is linked to any chapter test
    SELECT COUNT(UserChapter.id) > 0
    INTO showAnswer
    FROM "TestAttempt"
    JOIN "Test" ON "Test".id = "TestAttempt"."testId"
               AND "TestAttempt"."id" = usertestattemptid
               AND "TestAttempt"."completed" = TRUE
    LEFT JOIN "ChapterTest" ON "ChapterTest"."testId" = "Test".id
                           AND "Test".free = FALSE
    LEFT JOIN (
        SELECT 1 AS "id", "User".id AS "userId",
               "Topic".id AS "chapterId",
               "Subject".id AS "subjectId"
        FROM "User"
        JOIN "UserCourse" ON "User".id = "UserCourse"."userId"
        JOIN "Subject" ON "UserCourse"."courseId" = "Subject"."courseId"
        JOIN "SubjectChapter" ON "Subject".id = "SubjectChapter"."subjectId"
        JOIN "Topic" ON "SubjectChapter"."chapterId" = "Topic".id
        WHERE "UserCourse"."expiryAt" >= NOW()
          AND "SubjectChapter".deleted = FALSE
    ) UserChapter ON UserChapter."chapterId" = "ChapterTest"."chapterId"
                 AND UserChapter."userId" = "TestAttempt"."userId"
    GROUP BY "TestAttempt".id;

    RETURN showAnswer;
END;
$$;

alter function "TestAttemptDetailFunc"(integer) owner to learner;
