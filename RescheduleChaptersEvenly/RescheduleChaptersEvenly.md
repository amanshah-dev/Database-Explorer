create function "RescheduleChaptersEvenly"(p_userid integer, p_userscheduleid bigint, newchapters bigint[]) 
    returns void 
    language plpgsql 
as 
$$
DECLARE 
    futureTestDates DATE[]; 
    testScheduleId BIGINT; 
    chapterIndex INT := 1; 
    chaptersPerTest INT; 
    extraChapters INT; 
    currentTestIndex INT := 1; 
    seqId INT; 
    p_chapterId BIGINT; 
    totalChaptersCount CONSTANT INTEGER := 96; -- Total expected chapters 
    numTests INT; 
BEGIN 
    -- Fetch all future test dates for the given userScheduleId 
    SELECT array_agg("testDate") INTO futureTestDates 
    FROM "TestSchedule" 
    WHERE "userScheduleId" = p_userScheduleId 
    AND "userId" = p_userId 
    AND "testDate" > CURRENT_DATE; 

    -- Check if there are future TestSchedules for the provided userScheduleId and userId 
    IF array_length(futureTestDates, 1) IS NULL THEN 
        RAISE EXCEPTION 'No future TestSchedules found for user % and userScheduleId %', p_userId, p_userScheduleId; 
    END IF; 

    -- Delete all existing TestScheduleChapter entries for the given userScheduleId 
    DELETE FROM "TestScheduleChapter" 
    WHERE "testScheduleId" IN ( 
        SELECT "id" FROM "TestSchedule" WHERE "userScheduleId" = p_userScheduleId AND "userId" = p_userId 
    ) 
    AND "userId" = p_userId; 

    -- Delete all TestSchedules for the given userScheduleId and userId 
    DELETE FROM "TestSchedule" 
    WHERE "userScheduleId" = p_userScheduleId 
    AND "userId" = p_userId; 

    -- Calculate number of tests 
    numTests := array_length(futureTestDates, 1); 

    -- Calculate how many chapters per test and handle any extra chapters 
    chaptersPerTest := array_length(newChapters, 1) / numTests; 
    extraChapters := array_length(newChapters, 1) % numTests; 

    -- Loop through each future test date and create new TestSchedules 
    FOR currentTestIndex IN 1..numTests LOOP 
        -- Insert into TestSchedule 
        INSERT INTO "TestSchedule" ( 
            "userScheduleId", "startDate", "endDate", "testDate", "createdAt", "updatedAt", 
            "previouslyDone", "userId", "testNo" 
        ) VALUES ( 
            p_userScheduleId, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, futureTestDates[currentTestIndex], 
            CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, FALSE, p_userId, currentTestIndex 
        ) RETURNING "id" INTO testScheduleId; 

        seqId := 1; -- Resetting sequence ID for each TestSchedule 

        -- Insert chaptersPerTest chapters into the TestSchedule 
        FOR i IN 1..chaptersPerTest LOOP 
            IF chapterIndex > array_length(newChapters, 1) THEN 
                EXIT; -- No more chapters to distribute 
            END IF; 

            -- Get the chapter ID from the array 
            p_chapterId := newChapters[chapterIndex]; 

            -- Insert the chapter into the TestScheduleChapter table 
            INSERT INTO "TestScheduleChapter" ( 
                "testScheduleId", "chapterId", "seqId", "completed", "partial", "userId" 
            ) VALUES ( 
                testScheduleId, p_chapterId, seqId, FALSE, FALSE, p_userId 
            ); 

            -- Increment counters 
            chapterIndex := chapterIndex + 1; 
            seqId := seqId + 1; 
        END LOOP; 

        -- Distribute extra chapters (if any) 
        IF currentTestIndex <= extraChapters AND chapterIndex <= array_length(newChapters, 1) THEN 
            p_chapterId := newChapters[chapterIndex]; 

            INSERT INTO "TestScheduleChapter" ( 
                "testScheduleId", "chapterId", "seqId", "completed", "partial", "userId" 
            ) VALUES ( 
                testScheduleId, p_chapterId, seqId, FALSE, FALSE, p_userId 
            ); 

            chapterIndex := chapterIndex + 1; 
        END IF; 
    END LOOP; 

    -- Handle partial chapter selections if needed 
    IF array_length(newChapters, 1) < totalChaptersCount THEN 
        PERFORM "HandlePartialChapterSelection"(p_userScheduleId, newChapters, p_userId); 
    END IF; 

    RAISE NOTICE 'Chapters rescheduled successfully for userId % and userScheduleId %', p_userId, p_userScheduleId; 

EXCEPTION 
    WHEN OTHERS THEN 
        RAISE NOTICE 'Failed to reschedule chapters due to: %', SQLERRM; 
        RAISE; 
END; 
$$;

alter function "RescheduleChaptersEvenly"(integer, bigint, bigint[]) owner to neetprep_rw;
