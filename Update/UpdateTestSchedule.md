create function "UpdateTestSchedule"(p_testscheduleid bigint, p_userid integer, p_testid bigint, p_testattemptid bigint, p_testscore bigint) returns void
    language plpgsql
as
$$
BEGIN
    UPDATE "TestSchedule"
    SET 
        "testId" = p_testId,
        "testAttemptId" = p_testAttemptId,
        "testScore" = p_testScore,
        "updatedAt" = CURRENT_TIMESTAMP
    WHERE "id" = p_testScheduleId AND "userId" = p_userId;
END;
$$;

alter function "UpdateTestSchedule"(bigint, integer, bigint, bigint, bigint) owner to neetprep_rw;
