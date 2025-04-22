create function schedule_streak_interruption_notifications(inactivitydays integer DEFAULT 1) 
    returns void 
    language plpgsql 
as 
$$
BEGIN
    -- Call the PopulateDailyUserEvent function
    PERFORM public."PopulateDailyUserEvent"();

    WITH alreadyScheduleNotifi AS (
        SELECT DISTINCT
            "userId"
        FROM
            "Notification"
        WHERE
            "contextType" = 'StreakInterruption'
            AND "scheduledAt" >= CURRENT_DATE - INTERVAL '5.5 hours'
            AND "scheduledAt" < CURRENT_DATE + 1 - INTERVAL '5.5 hours'
            AND id > 312384763 AND "scheduledAt" IS NOT NULL
    ),
    inactiveUsers AS (
        SELECT
            "userId",
            ("eventInfo"->>'firstAnswerTime')::TIMESTAMPTZ AT TIME ZONE 'UTC' AS firstAnswerTimeUTC
        FROM "DailyUserEvent"
        WHERE "event" = 'Question' AND "eventDate" = (CURRENT_DATE - INTERVAL '5.5 hours')::DATE - inactivitydays - 1
        AND (NOW()::DATE - ("eventInfo"->>'firstAnswerTime')::DATE) - 1 = inactivitydays
    )

    -- Insert notifications for inactive users
    INSERT INTO "Notification" ("userId", "title", "body", "scheduledAt", "contextType")
    SELECT
        iu."userId",
        'Streak has been interrupted',
        'You have not been active for the past ' || inactivitydays || ' days. Come start solving questions to continue your streak!',
        CURRENT_DATE + (iu.firstAnswerTimeUTC - DATE_TRUNC('day', iu.firstAnswerTimeUTC))::INTERVAL,
        'StreakInterruption'
    FROM
        inactiveUsers iu
    WHERE
        iu."userId" NOT IN (SELECT "userId" FROM alreadyScheduleNotifi)
    ON CONFLICT ("scheduledAt", "userId", "contextId", "contextType")
    WHERE "id" > 312384763 AND "scheduledAt" IS NOT NULL
    DO NOTHING;

END;
$$;

alter function schedule_streak_interruption_notifications(integer) owner to learner;
