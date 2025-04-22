create function "UpdateTestScheduleChapter"(p_userid integer, chapters jsonb) returns void
    language plpgsql
as
$$
DECLARE
    originalTestScheduleId BIGINT;
    originalSeqId INT;
    chapter jsonb;
    p_chapterId BIGINT;
    newTestScheduleId BIGINT;
    newPosition INT;
    p_completed BOOLEAN;
BEGIN
    FOR chapter IN SELECT * FROM jsonb_array_elements(chapters)
    LOOP
        p_chapterId := (chapter->>'chapter_id')::BIGINT;
        newTestScheduleId := (chapter->>'new_test_schedule_id')::BIGINT;
        newPosition := (chapter->>'new_position')::INT;
        p_completed := (chapter->>'completed')::BOOLEAN;

        -- Ensure the chapter belongs to the user
        SELECT "testScheduleId", "seqId" INTO originalTestScheduleId, originalSeqId
        FROM "TestScheduleChapter"
        JOIN "TestSchedule" ON "TestSchedule"."id" = "TestScheduleChapter"."testScheduleId"
        JOIN "UserSchedule" ON "UserSchedule"."id" = "TestSchedule"."userScheduleId"
        WHERE "UserSchedule"."userId" = p_userId AND "TestScheduleChapter"."chapterId" = p_chapterId;

        IF FOUND THEN
            -- Remove the chapter from the original test schedule only for the specific user
            DELETE FROM "TestScheduleChapter" 
            WHERE "chapterId" = p_chapterId AND "userId" = p_userId;

            -- Decrement seqIds for chapters that were after this chapter in the original schedule, scoped by user ID
            UPDATE "TestScheduleChapter"
            SET "seqId" = "seqId" - 1
            WHERE "testScheduleId" = originalTestScheduleId AND "seqId" > originalSeqId AND "userId" = p_userId;

            -- Adjust seqIds for chapters in the new test schedule to make space for the new chapter, scoped by user ID
            UPDATE "TestScheduleChapter"
            SET "seqId" = "seqId" + 1
            WHERE "testScheduleId" = newTestScheduleId AND "seqId" >= newPosition AND "userId" = p_userId;

            -- Insert the chapter into the new test schedule at the specified newPosition, ensuring it's associated with the right user
            INSERT INTO "TestScheduleChapter" ("testScheduleId", "chapterId", "seqId", "completed", "partial", "userId")
            VALUES (newTestScheduleId, p_chapterId, newPosition, p_completed, FALSE, p_userId);

        ELSE
            RAISE EXCEPTION 'Chapter not found or not accessible by the user.';
        END IF;
    END LOOP;

EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Failed to update TestScheduleChapter due to: %', SQLERRM;
        RAISE;
END;
$$;

alter function "UpdateTestScheduleChapter"(integer, jsonb) owner to neetprep_rw;
