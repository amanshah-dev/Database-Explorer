CREATE FUNCTION delete_related_schedules_for_user(p_userid INTEGER) 
    RETURNS VOID
    LANGUAGE plpgsql
AS
$$
BEGIN
    -- Delete entries from TestScheduleChapter where testScheduleId matches those found in TestSchedule for the provided userId
    DELETE FROM "TestScheduleChapter"
    WHERE "testScheduleId" IN (
        SELECT "id" FROM "TestSchedule"
        WHERE "userScheduleId" IN (
            SELECT "id" FROM "UserSchedule"
            WHERE "userId" = p_userId AND "deleted" = TRUE
        )
    );

    -- Delete entries from TestSchedule related to the provided userId
    DELETE FROM "TestSchedule"
    WHERE "userScheduleId" IN (
        SELECT "id" FROM "UserSchedule"
        WHERE "userId" = p_userId AND "deleted" = TRUE
    );

    -- Optionally, delete from UserSchedule as well, uncomment if necessary
    -- DELETE FROM "UserSchedule"
    -- WHERE "userId" = p_userId AND "deleted" = TRUE;
END;
$$;

ALTER FUNCTION delete_related_schedules_for_user(INTEGER) OWNER TO neetprep_rw;
