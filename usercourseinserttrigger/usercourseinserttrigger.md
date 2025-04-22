create function usercourseinserttrigger() returns trigger
    language plpgsql
as
$$
DECLARE
    existing_count INTEGER;
    longest_entry RECORD;
    new_duration INTERVAL;
    start_time TIMESTAMP;
    end_time TIMESTAMP;
BEGIN
    -- Step 1: Check if the courseId matches the specific course (2135)
    IF NEW."courseId" != 2135 THEN
        RAISE NOTICE 'Skipping trigger for courseId: %. Only courseId 2135 is processed.', NEW."courseId";
        RETURN NEW; -- Proceed with the original INSERT
    END IF;

    -- Step 2: Validate inputs from NEW record
    RAISE NOTICE 'Step 2: Validating inputs for userId: %, courseId: %, startedAt: %, expiryAt: %',
        NEW."userId", NEW."courseId", NEW."startedAt", NEW."expiryAt";
    IF NEW."userId" IS NULL OR NEW."courseId" IS NULL OR NEW."startedAt" IS NULL OR NEW."expiryAt" IS NULL THEN
        RAISE NOTICE 'Invalid input: userId, courseId, startedAt, and expiryAt must not be null.';
        RETURN NULL; -- Skip the INSERT
    END IF;
    IF NEW."expiryAt" <= NEW."startedAt" THEN
        RAISE NOTICE 'Invalid input: expiryAt (%) must be after startedAt (%).', NEW."expiryAt", NEW."startedAt";
        RETURN NULL; -- Skip the INSERT
    END IF;
    RAISE NOTICE 'Input validation passed.';

    -- Step 3: Check if this is the first entry for the user in the course
    RAISE NOTICE 'Step 3: Checking for existing entries for userId: % in courseId: %.', NEW."userId", NEW."courseId";
    start_time := clock_timestamp();
    SELECT COUNT(*)
    INTO existing_count
    FROM "UserCourse"
    WHERE "userId" = NEW."userId" AND "courseId" = NEW."courseId";
    end_time := clock_timestamp();
    RAISE NOTICE 'Step 3 Complete: Found % entries in % seconds.',
        existing_count, EXTRACT(EPOCH FROM (end_time - start_time));

    IF existing_count = 0 THEN
        RAISE NOTICE 'No existing entries found. Proceeding with original INSERT.';
        RETURN NEW; -- Proceed with the original INSERT
    END IF;

    -- Step 4: Find the longest-duration non-expired entry
    RAISE NOTICE 'Step 4: Finding the longest-duration non-expired entry for userId: % in courseId: %.', NEW."userId", NEW."courseId";
    start_time := clock_timestamp();
    SELECT
        uc."startedAt",
        uc."expiryAt",
        (uc."expiryAt" - uc."startedAt") AS duration
    INTO longest_entry
    FROM "UserCourse" uc
    WHERE
        uc."userId" = NEW."userId"
        AND uc."courseId" = NEW."courseId"
        AND uc."expiryAt" > CURRENT_TIMESTAMP
    ORDER BY (uc."expiryAt" - uc."startedAt") DESC
    LIMIT 1;
    end_time := clock_timestamp();
    RAISE NOTICE 'Step 4 Complete: Found longest-duration entry in % seconds.', EXTRACT(EPOCH FROM (end_time - start_time));

    -- Step 5: Decide whether to modify the NEW record
    IF longest_entry IS NULL THEN
        RAISE NOTICE 'No non-expired entries found. Proceeding with original INSERT.';
        RETURN NEW; -- Proceed with the original INSERT
    END IF;

    RAISE NOTICE 'Longest-duration entry: startedAt: %, expiryAt: %, duration: %.',
        longest_entry."startedAt", longest_entry."expiryAt", longest_entry.duration;

    -- Step 6: Check for overlap and compute new dates
    RAISE NOTICE 'Step 6: Checking for overlap between new entry (startedAt: %, expiryAt: %) and longest-duration entry.',
        NEW."startedAt", NEW."expiryAt";
    IF NEW."startedAt" > longest_entry."expiryAt" THEN
        RAISE NOTICE 'No overlap: New startedAt (%) is after longest entry expiryAt (%). Proceeding with original INSERT.',
            NEW."startedAt", longest_entry."expiryAt";
        RETURN NEW; -- Proceed with the original INSERT
    ELSE
        RAISE NOTICE 'Overlap detected: New startedAt (%) is before or equal to longest entry expiryAt (%). Extending duration.',
            NEW."startedAt", longest_entry."expiryAt";
        -- Compute the duration of the new entry
        new_duration := NEW."expiryAt" - NEW."startedAt";
        RAISE NOTICE 'New entry duration: %.', new_duration;

        -- Modify the NEW record with U1's startedAt and extended expiryAt
        NEW."startedAt" := longest_entry."startedAt";
        NEW."expiryAt" := longest_entry."expiryAt" + new_duration;
        NEW."updatedAt" := CURRENT_TIMESTAMP;
        RAISE NOTICE 'Modified NEW record: startedAt: %, expiryAt: %.', NEW."startedAt", NEW."expiryAt";
        RETURN NEW; -- Proceed with the modified INSERT
    END IF;
EXCEPTION WHEN OTHERS THEN
    RAISE NOTICE 'Error in trigger for userId: % in courseId: %: %', NEW."userId", NEW."courseId", SQLERRM;
    RETURN NULL; -- Skip the INSERT
END;
$$;

alter function usercourseinserttrigger() owner to learner;
