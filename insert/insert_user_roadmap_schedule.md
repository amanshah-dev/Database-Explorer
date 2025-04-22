CREATE FUNCTION insert_user_roadmap_schedule(
    schedule_name TEXT, 
    schedule_description TEXT, 
    tests JSONB, 
    topics JSONB, 
    user_id INTEGER
) RETURNS VOID
LANGUAGE plpgsql
AS
$$
DECLARE
    schedule_id INTEGER;
    schedule_item_id INTEGER;
    schedule_item_chapter_id INTEGER;
    topic_var_id INTEGER;
    test_id INTEGER;
    asset_link TEXT;
    video_lecture_link TEXT;
    practice_dpp_link TEXT;
    high_yield_mcq_link TEXT;
    short_video_link TEXT;
    subject_id INTEGER;
    mapped_topic_id INTEGER;
    mapped_subject_id INTEGER;
    topic_test_mapping JSONB;
    topic_to_subject_mapping JSONB;
    mapped_topics JSONB;
    test JSONB;
BEGIN
    -- Step 1: Retrieve the mapping of topicId and testId (for highYieldMCQ)
    SELECT jsonb_agg(jsonb_build_object('topicId', rt."chapterId", 'testId', rt."id"))
    INTO topic_test_mapping
    FROM (
        WITH RecommendedTests AS (
            SELECT t."id",
                   cqs."chapterId",
                   t."createdAt"
            FROM "Test" t
            JOIN "ChapterQuestionSet" cqs ON cqs."testId" = t."id"
            WHERE cqs."chapterId" IN (
                SELECT "id" FROM "Topic" WHERE "subjectId" IN (53, 54, 55, 56)
            )
            AND t."name" ILIKE '%Recommended%'
        ),
        LatestTests AS (
            SELECT DISTINCT ON (rt."chapterId") rt."chapterId", rt."id"
            FROM RecommendedTests rt
            ORDER BY rt."chapterId", rt."createdAt" DESC
        )
        SELECT * FROM LatestTests
    ) AS rt;

    -- Step 2: Retrieve the topicId to subjectId mapping (for videoLecture)
    SELECT jsonb_agg(jsonb_build_object('topicId', t."id", 'subjectId', t."subjectId"))
    INTO topic_to_subject_mapping
    FROM "Topic" t
    WHERE t."subjectId" IN (53, 54, 55, 56);

    -- Step 3: Retrieve the mapped topics from Course 8 and Course 29 (for shortVideo)
    SELECT jsonb_agg(jsonb_build_object(
        'originalTopicId', mt."originalTopicId",
        'originalSubjectId', mt."originalSubjectId",
        'mappedTopicId', mt."mappedTopicId",
        'mappedSubjectId', mt."mappedSubjectId"))
    INTO mapped_topics
    FROM (
        WITH Course8Topics AS (
            SELECT "CC"."chapterId" AS "topicId", "T"."subjectId"
            FROM "CourseChapter" AS "CC"
            JOIN "Topic" AS "T" ON "CC"."chapterId" = "T"."id"
            WHERE "CC"."courseId" = 8
        ),
        Course29Topics AS (
            SELECT "CC"."chapterId" AS "topicId", "T"."subjectId"
            FROM "CourseChapter" AS "CC"
            JOIN "Topic" AS "T" ON "CC"."chapterId" = "T"."id"
            WHERE "CC"."courseId" = 29
        ),
        MappedTopics AS (
            SELECT
                C8."topicId" AS "originalTopicId",
                C8."subjectId" AS "originalSubjectId",
                DC."dupId" AS "mappedTopicId",
                C29."subjectId" AS "mappedSubjectId"
            FROM Course8Topics C8
            JOIN "DuplicateChapter" DC ON C8."topicId" = DC."origId"
            JOIN Course29Topics C29 ON DC."dupId" = C29."topicId"
        )
        SELECT * FROM MappedTopics
    ) AS mt;

    -- Step 4: Insert the new Schedule
    INSERT INTO "Schedule" ("createdAt", "updatedAt", name, description, "isActive", "isScoreBooster", "userId")
    VALUES (CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, schedule_name, schedule_description, false, true, user_id)
    RETURNING id INTO schedule_id;

    -- Step 5: Iterate over the tests and insert ScheduleItems
    FOR test IN SELECT * FROM jsonb_array_elements(tests) LOOP
        INSERT INTO "ScheduleItem" ("createdAt", "updatedAt", name, "scheduleId", hours, "scheduledAt")
        VALUES (
            CURRENT_TIMESTAMP,
            CURRENT_TIMESTAMP,
            test->>'name',
            schedule_id,
            3,
            (test->>'scheduledAt')::timestamp with time zone
        )
        RETURNING id INTO schedule_item_id;

        -- Step 6: Insert ScheduleItemChapters for topics linked to each ScheduleItem
        FOR topic_var_id IN SELECT * FROM jsonb_array_elements_text(test->'topicIds') LOOP
            INSERT INTO "ScheduleItemChapter" ("topicId", "scheduleItemId", deleted, "createdAt", "updatedAt")
            VALUES (
                topic_var_id::integer,
                schedule_item_id,
                false,
                CURRENT_TIMESTAMP,
                CURRENT_TIMESTAMP
            )
            RETURNING id INTO schedule_item_chapter_id;

            -- Insert ScheduleItemAssets (Diagnostic Test)
            SELECT (topic->>'testId')::integer INTO test_id
            FROM jsonb_array_elements(topics) AS topic
            WHERE (topic->>'topicId')::integer = topic_var_id::integer
            LIMIT 1;

            asset_link := 'https://www.neetprep.com/neet-test/' || test_id;
            INSERT INTO "ScheduleItemAsset" ("createdAt", "updatedAt", "assetLink", "assetName", "scheduleItemChapterId")
            VALUES (CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, asset_link, 'diagnosticTest', schedule_item_chapter_id);

            -- Insert videoLecture asset
            SELECT t->>'subjectId' INTO subject_id
            FROM jsonb_array_elements(topic_to_subject_mapping) AS t
            WHERE (t->>'topicId')::integer = topic_var_id;

            video_lecture_link := 'https://www.neetprep.com/video-classes/' || subject_id || '/' || topic_var_id;
            INSERT INTO "ScheduleItemAsset" ("createdAt", "updatedAt", "assetLink", "assetName", "scheduleItemChapterId")
            VALUES (CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, video_lecture_link, 'videoLecture', schedule_item_chapter_id);

            -- Insert practiceDPP asset
            practice_dpp_link := 'https://www.neetprep.com/dpps/create?chapter_ids=[' || topic_var_id || ']';
            INSERT INTO "ScheduleItemAsset" ("createdAt", "updatedAt", "assetLink", "assetName", "scheduleItemChapterId")
            VALUES (CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, practice_dpp_link, 'practiceDPP', schedule_item_chapter_id);

            -- Insert highYieldMCQ asset
            SELECT t->>'testId' INTO test_id
            FROM jsonb_array_elements(topic_test_mapping) AS t
            WHERE (t->>'topicId')::integer = topic_var_id;

            high_yield_mcq_link := 'https://www.neetprep.com/questions?testId=' || test_id;
            INSERT INTO "ScheduleItemAsset" ("createdAt", "updatedAt", "assetLink", "assetName", "scheduleItemChapterId")
            VALUES (CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, high_yield_mcq_link, 'highYieldMCQs', schedule_item_chapter_id);

            -- Insert shortVideo asset
            SELECT t->>'mappedTopicId', t->>'mappedSubjectId' INTO mapped_topic_id, mapped_subject_id
            FROM jsonb_array_elements(mapped_topics) AS t
            WHERE (t->>'originalTopicId')::integer = topic_var_id;

            IF mapped_topic_id IS NOT NULL THEN
                short_video_link := 'https://www.neetprep.com/video-classes/' || mapped_subject_id || '/' || mapped_topic_id || '?courseId=29';
                INSERT INTO "ScheduleItemAsset" ( "createdAt", "updatedAt", "assetLink", "assetName", "scheduleItemChapterId")
                VALUES (CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, short_video_link, 'shortVideo', schedule_item_chapter_id);
            END IF;
        END LOOP;
    END LOOP;

    -- Step 7: Insert into UserSchedule
    INSERT INTO "UserSchedule" ("userId", "startDate", "endDate", "name", "deleted", "externalScheduleId")
    VALUES (
        user_id,
        CURRENT_TIMESTAMP,
        (SELECT MAX("scheduledAt") FROM "ScheduleItem" WHERE "scheduleId" = schedule_id),
        schedule_name,
        false,
        schedule_id
    );
END;
$$;

ALTER FUNCTION insert_user_roadmap_schedule(TEXT, TEXT, JSONB, JSONB, INTEGER) OWNER TO learner;
