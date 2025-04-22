create function schedule_daily_nudges(eventwindowhandler integer DEFAULT 0, scheduledatehandler integer DEFAULT 1) 
    returns void 
    language plpgsql 
as 
$$
BEGIN
    -- Call the PopulateDailyUserEvent function
    PERFORM public."PopulateDailyUserAllEvent"();

    WITH alreadyScheduleNotifi AS (
        SELECT DISTINCT
            "userId"
        FROM
            "Notification"
        WHERE
            "contextType" = 'DailyNudges'
            AND "scheduledAt" BETWEEN CURRENT_TIMESTAMP AND CURRENT_TIMESTAMP + INTERVAL '24 hours'
            AND id > 312384763 AND "scheduledAt" IS NOT NULL
    ),
    inactiveUsers AS (
        SELECT
            "userId",
            "eventInfo",
            "eventDate"
        FROM
            "DailyUserEvent"
        WHERE
            "event" = 'Question'
            AND ("eventDate" = CURRENT_DATE or "eventDate" = CURRENT_DATE - 1 - eventWindowHandler)
            AND ("eventInfo"->>'firstAnswerTime') IS NOT NULL
    )

    -- Insert notifications for inactive users
    INSERT INTO "Notification" ("userId", "title", "body", "scheduledAt", "contextType")
    SELECT
        iu."userId",
        'Daily Nudge',
        'Hey its Time to Practice: Complete the Daily Practice Goal to maintain the Streak',
        (DATE(iu."eventInfo"->>'firstAnswerTime') + scheduledatehandler ) + CAST("eventInfo"->>'firstAnswerTime' AS TIMESTAMPTZ)::TIME ,
        'DailyNudges'
    FROM
        inactiveUsers iu
    WHERE
        iu."userId" NOT IN (SELECT "userId" FROM alreadyScheduleNotifi)
    ON CONFLICT ("scheduledAt", "userId", "contextId", "contextType")
    WHERE "id" > 312384763 AND "scheduledAt" IS NOT NULL
    DO NOTHING;

END;
$$;

alter function schedule_daily_nudges(integer, integer) owner to learner;
