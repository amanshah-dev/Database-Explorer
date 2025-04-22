CREATE FUNCTION "GenerateUserSchedule"(p_scheduleid BIGINT, p_chapterids BIGINT[], p_userid INTEGER) 
    RETURNS VOID
    LANGUAGE plpgsql
AS
$$
DECLARE
    userScheduleId            BIGINT;
    testDates                 DATE[];
    totalChapters             INTEGER;
    numTests                  INTEGER;
    chaptersPerTest           INTEGER;
    extraChapters             INTEGER;
    testId                    BIGINT;
    scheduleEndDate           TIMESTAMP;
    seqId                     INTEGER;
    chapterIndex              INTEGER := 1;
    testNo                    INTEGER := 1;
    originalChapterIdsIsNull  BOOLEAN := FALSE;
    shuffledChapterIds        BIGINT[];
BEGIN
    -- Check if the user already has a schedule
    RAISE NOTICE 'Checking if user already has a schedule for schedule ID % and user ID %.', p_scheduleID, p_userId;
    SELECT "id"
    INTO userScheduleId
    FROM "UserSchedule"
    WHERE "scheduleId" = p_scheduleID
      AND "userId" = p_userId
      AND "deleted" = FALSE;

    IF userScheduleId IS NOT NULL THEN
        RAISE NOTICE 'User already exists with schedule ID % and user ID %.', p_scheduleID, p_userId;
        RETURN;
    END IF;

    -- Retrieve schedule end date
    RAISE NOTICE 'Fetching schedule end date for schedule ID %.', p_scheduleID;
    SELECT "endDate" INTO scheduleEndDate FROM "Scheduler" WHERE "id" = p_scheduleID;
    IF scheduleEndDate IS NULL THEN
        scheduleEndDate := '2025-05-30'; -- Default end date
        RAISE NOTICE 'Schedule end date is null, setting default end date to %.', scheduleEndDate;
    END IF;

    -- Create user schedule
    RAISE NOTICE 'Creating user schedule for user ID % and schedule ID %.', p_userId, p_scheduleID;
    INSERT INTO "UserSchedule" ("scheduleId", "userId", "startDate", "endDate", "name", "deleted")
    VALUES (p_scheduleID, p_userId, CURRENT_TIMESTAMP, scheduleEndDate, 'Neetprep Scheduler', FALSE)
    RETURNING "id" INTO userScheduleId;
    RAISE NOTICE 'User schedule created with ID %.', userScheduleId;

    -- If no chapter IDs provided, fetch all chapters
    IF array_length(p_chapterIds, 1) IS NULL THEN
        originalChapterIdsIsNull := TRUE;
        RAISE NOTICE 'No chapter IDs provided, fetching all chapters.';
        SELECT array_agg("id") INTO p_chapterIds FROM "ChaptersScheduler";
        RAISE NOTICE 'Fetched chapters: %.', p_chapterIds;
    END IF;

    -- Validate chapter IDs
    RAISE NOTICE 'Validating chapter IDs to exclude NULL values.';
    p_chapterIds := array_remove(p_chapterIds, NULL);
    RAISE NOTICE 'Validated chapters: %.', p_chapterIds;

    -- Shuffle chapter IDs
    RAISE NOTICE 'Shuffling chapter IDs for random distribution.';
    SELECT array_agg("id") INTO shuffledChapterIds
    FROM (
        SELECT "id"
        FROM unnest(p_chapterIds) WITH ORDINALITY AS t("id", ord)
        ORDER BY random()
    ) subquery;
    RAISE NOTICE 'Shuffled chapter IDs: %.', shuffledChapterIds;

    p_chapterIds := shuffledChapterIds;

    -- Calculate the total number of chapters from the provided chapter IDs
    SELECT COUNT(*) INTO totalChapters
    FROM "ChaptersScheduler"
    WHERE "id" = ANY (p_chapterIds);
    RAISE NOTICE 'Total chapters to distribute: %.', totalChapters;

    -- Generate test dates using the testDateGenFunc
    RAISE NOTICE 'Generating test dates.';
    SELECT testDateGenFunc(CURRENT_TIMESTAMP::DATE, scheduleEndDate::DATE) INTO testDates;
    RAISE NOTICE 'Generated test dates: %.', testDates;

    -- Determine number of tests
    numTests := array_length(testDates, 1);
    RAISE NOTICE 'Number of tests to create: %.', numTests;

    -- Calculate chapters per test and handle any extra chapters
    chaptersPerTest := totalChapters / numTests;
    extraChapters := totalChapters % numTests;
    RAISE NOTICE 'Chapters per test: %, Extra chapters: %.', chaptersPerTest, extraChapters;

    -- Distribute chapters across tests
    FOR testIndex IN 1..numTests LOOP
        RAISE NOTICE 'Creating test schedule for test number %.', testIndex;

        -- Insert a new test schedule
        INSERT INTO "TestSchedule" (
            "userScheduleId", "startDate", "endDate", "testDate", "createdAt", "updatedAt", 
            "previouslyDone", "userId", "testNo"
        ) VALUES (
            userScheduleId, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, testDates[testIndex], 
            CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, FALSE, p_userId, testIndex
        ) RETURNING "id" INTO testId;
        RAISE NOTICE 'Created test schedule with ID % for test number %.', testId, testIndex;

        seqId := 1; -- Resetting sequence ID for each TestSchedule

        -- Insert chaptersPerTest chapters into the TestSchedule
        FOR i IN 1..chaptersPerTest LOOP
            IF chapterIndex > totalChapters THEN
                RAISE NOTICE 'No more chapters to distribute, exiting loop.';
                EXIT;
            END IF;

            RAISE NOTICE 'Assigning chapter ID % to test schedule ID %.', p_chapterIds[chapterIndex], testId;

            -- Ensure chapter ID is not NULL
            IF p_chapterIds[chapterIndex] IS NOT NULL THEN
                INSERT INTO "TestScheduleChapter" (
                    "testScheduleId", "chapterId", "seqId", "completed", "partial", "userId"
                ) VALUES (
                    testId, p_chapterIds[chapterIndex], seqId, FALSE, FALSE, p_userId
                );
                chapterIndex := chapterIndex + 1;
                seqId := seqId + 1;
            ELSE
                RAISE NOTICE 'Skipping NULL chapter ID at index %.', chapterIndex;
                chapterIndex := chapterIndex + 1;
            END IF;
        END LOOP;

        -- Distribute extra chapters (if any)
        IF testIndex <= extraChapters AND chapterIndex <= totalChapters THEN
            IF p_chapterIds[chapterIndex] IS NOT NULL THEN
                RAISE NOTICE 'Assigning extra chapter ID % to test schedule ID %.', p_chapterIds[chapterIndex], testId;
                INSERT INTO "TestScheduleChapter" (
                    "testScheduleId", "chapterId", "seqId", "completed", "partial", "userId"
                ) VALUES (
                    testId, p_chapterIds[chapterIndex], seqId, FALSE, FALSE, p_userId
                );
                chapterIndex := chapterIndex + 1;
            ELSE
                RAISE NOTICE 'Skipping NULL extra chapter ID at index %.', chapterIndex;
                chapterIndex := chapterIndex + 1;
            END IF;
        END IF;
    END LOOP;

    -- Handle partial chapter selections if original chapter IDs were not null
    IF NOT originalChapterIdsIsNull THEN
        RAISE NOTICE 'Handling partial chapter selection for user schedule ID %.', userScheduleId;
        PERFORM "HandlePartialChapterSelection"(userScheduleId, p_chapterIds, p_userId);
    END IF;

    RAISE NOTICE 'Schedule created successfully for userId % and scheduleId %.', p_userId, p_scheduleID;

EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Failed to generate user schedule due to: %', SQLERRM;
        RAISE;
END;
$$;

ALTER FUNCTION "GenerateUserSchedule"(BIGINT, BIGINT[], INTEGER) OWNER TO neetprep_rw;
