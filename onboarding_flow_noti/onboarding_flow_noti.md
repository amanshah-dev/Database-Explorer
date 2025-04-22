CREATE FUNCTION onboarding_flow_noti(
    body_param TEXT DEFAULT 'Hi <name>, come to NEETprep for a quick NEET practice'::TEXT, 
    eventwindowhandler INTEGER DEFAULT 0, 
    scheduledatehandler INTEGER DEFAULT 1
) RETURNS VOID
LANGUAGE plpgsql
AS
$$
BEGIN
WITH inactiveUsers AS (
    SELECT DISTINCT du1."userId", du1."eventInfo", du1."eventDate"
    FROM "DailyUserEvent" du1
    LEFT JOIN "Notification" n ON du1."userId" = n."userId"
        AND n."contextType" = 'Onboarding Nudge'
        AND n."scheduledAt" BETWEEN (CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata')::date 
            AND ((CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata')::date + 1)
        AND n.id > 312384763 
        AND n."scheduledAt" IS NOT NULL
    WHERE du1."event" = 'Question'
        AND du1."eventDate" = ((CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata')::date - 1 - eventwindowhandler)
        AND ("eventInfo"->>'firstAnswerTime') IS NOT NULL
        AND NOT EXISTS (
            SELECT 1
            FROM "DailyUserEvent" du2
            WHERE du1."userId" = du2."userId"
                AND du2."event" = 'Question'
                AND du2."eventDate" > ((CURRENT_TIMESTAMP AT TIME ZONE 'Asia/Kolkata')::date - 1 -  eventwindowhandler)
        )
        AND n."userId" IS NULL
)
    -- Insert notifications for inactive users
    INSERT INTO "Notification" (
        "userId",
        "title",
        "body",
        "scheduledAt",
        "contextType"
    )
    SELECT iu."userId",
           'Practice NEET Questions Now',
           REPLACE(body_param, '<name>', COALESCE(up."firstName", 'there')),
           ("eventInfo"->>'firstAnswerTime')::timestamptz + (scheduledatehandler * 86400 * INTERVAL '1 second'),
           'Onboarding Nudge'
    FROM inactiveUsers iu
    JOIN "UserProfile" up ON iu."userId" = up."userId"
    ON CONFLICT ("scheduledAt", "userId", "contextId", "contextType")
    WHERE id > 312384763 AND "scheduledAt" IS NOT NULL
    DO NOTHING;
END;
$$;

ALTER FUNCTION onboarding_flow_noti(TEXT, INTEGER, INTEGER) OWNER TO learner;
