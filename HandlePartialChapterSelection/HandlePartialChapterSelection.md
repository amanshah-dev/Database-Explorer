CREATE FUNCTION "HandlePartialChapterSelection"(
    p_userscheduleid BIGINT, 
    p_chapterids BIGINT[], 
    p_userid INTEGER
) 
RETURNS VOID 
    LANGUAGE plpgsql 
AS
$$
DECLARE
    totalChaptersCount CONSTANT INTEGER := 96; -- Total expected chapters
    missingChapters BIGINT[];
    newTestScheduleId BIGINT;
    yesterday DATE;
    scheduleEndDate DATE;
BEGIN
    -- Check if all chapters are selected
    IF array_length(p_chapterIds, 1) = totalChaptersCount THEN
        RAISE NOTICE 'All chapters are selected, no action needed.';
        RETURN;
    END IF;

    -- Calculate yesterday's date
    yesterday := CURRENT_DATE - 1;

    -- Find chapters that are not included in the input
    SELECT array_agg("id") INTO missingChapters
    FROM "ChaptersScheduler"
    WHERE "id" <> ALL(p_chapterIds);

    -- Get the end date from the Scheduler table
    SELECT "endDate" INTO scheduleEndDate
    FROM "UserSchedule"
    WHERE "id" = p_userScheduleId;

    -- Insert a new TestSchedule with a previous date
    INSERT INTO "TestSchedule" (
        "userScheduleId",
        "startDate",
        "endDate",
        "testDate",
        "createdAt",
        "updatedAt",
        "previouslyDone",
        "userId")  -- Added userId to the TestSchedule insertion
    VALUES (
        p_userScheduleId,
        yesterday,
        scheduleEndDate,
        yesterday,
        CURRENT_TIMESTAMP,
        CURRENT_TIMESTAMP,
        TRUE,
        p_userId)  -- Using the passed userId
    RETURNING "id" INTO newTestScheduleId;

    -- Insert missing chapters into TestScheduleChapter with previouslyDone = true and including the userId
    FOR i IN 1..array_length(missingChapters, 1) LOOP
        INSERT INTO "TestScheduleChapter" (
            "testScheduleId",
            "chapterId",
            "seqId",
            "completed",
            "partial",
            "userId")  -- Added userId to the TestScheduleChapter insertion
        VALUES (
            newTestScheduleId,
            missingChapters[i],
            i,
            TRUE,
            FALSE,
            p_userId);  -- Using the passed userId
    END LOOP;

    RAISE NOTICE 'Missing chapters processed for user schedule ID % with new test schedule ID %.', p_userScheduleId, newTestScheduleId;

EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Failed to handle partial chapter selection due to: %', SQLERRM;
        RAISE;
END;
$$;

ALTER FUNCTION "HandlePartialChapterSelection"(bigint, bigint[], integer) OWNER TO neetprep_rw;
