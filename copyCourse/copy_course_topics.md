create function copy_course_topics(course_id integer) returns void
    language plpgsql
as
$$
DECLARE
    old_topic RECORD;
    new_topic_id INT;
    old_topic_ids INT[];
BEGIN
    -- Initialize the array to keep track of old topic IDs
    old_topic_ids := ARRAY(
        SELECT "chapterId"
        FROM "CourseChapter"
        WHERE "courseId" = course_id
    );

    -- Cursor to iterate over each topic of the given course
    FOR old_topic IN
        SELECT "t"."id" AS chapter_id, "cc"."subjectId" AS subject_id
        FROM "CourseChapter" AS "cc"
        JOIN "Topic" AS "t" ON "cc"."chapterId" = "t"."id"
        WHERE "cc"."courseId" = course_id
        ORDER BY t."subjectId" ASC, t."seqId" ASC, t."id" ASC
    LOOP
        -- Insert a new topic row and get the new topic ID
        INSERT INTO "Topic" ("published", "free", "subjectId", "position", "description", "image", "name", "sectionReady", "isComingSoon", "importUrl", "seqId")
        SELECT "published", "free", old_topic.subject_id, "position", "description", "image", "name", "sectionReady", "isComingSoon", "importUrl", "seqId"
        FROM "Topic"
        WHERE "id" = old_topic.chapter_id
        RETURNING "id" INTO new_topic_id;

        -- Update the existing SubjectChapter with the new topic ID
        UPDATE "SubjectChapter"
        SET "chapterId" = new_topic_id, "updatedAt" = NOW()
        WHERE "chapterId" = old_topic.chapter_id AND "subjectId" = old_topic.subject_id;

        -- Handle existing duplicates for the old topic ID
        WITH existing_duplicate AS (
            SELECT "id", "origId", "dupId"
            FROM "DuplicateChapter"
            WHERE "dupId" = old_topic.chapter_id
        )
        INSERT INTO "DuplicateChapter" ("origId", "dupId")
        SELECT ed."origId", new_topic_id
        FROM existing_duplicate ed
        ON CONFLICT ("origId", "dupId") DO NOTHING;

    END LOOP;
END;
$$;

alter function copy_course_topics(integer) owner to neetprep_rw;
