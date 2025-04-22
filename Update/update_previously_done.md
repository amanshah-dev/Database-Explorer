create function update_previously_done() returns trigger
    language plpgsql
as
$$
BEGIN
    -- Check if all chapters in the current test schedule are marked as completed
    IF (SELECT COUNT(*)
        FROM "TestScheduleChapter"
        WHERE "testScheduleId" = NEW."testScheduleId"
          AND "completed" = FALSE) = 0 THEN
        -- Update previouslyDone to true if all chapters are completed
        UPDATE "TestSchedule"
        SET "previouslyDone" = TRUE
        WHERE "id" = NEW."testScheduleId";
    END IF;
    RETURN NEW;
END;
$$;

alter function update_previously_done() owner to neetprep_rw;
