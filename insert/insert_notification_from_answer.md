CREATE FUNCTION insert_notification_from_answer(
    notification_title TEXT DEFAULT 'Daily Streaks Reminder'::TEXT, 
    notification_body TEXT DEFAULT 'Daily reminder to solve more question'::TEXT
) RETURNS VOID
LANGUAGE plpgsql
AS
$$
BEGIN
    INSERT INTO "Notification" ("userId", "contextId", "contextType", "title", "body", "scheduledAt")
    SELECT
        "Answer"."userId",
        MIN("Answer"."id"),
        'DailyPracticeReminder',
        notification_title,
        FORMAT(notification_body, COUNT("Answer"."id")),
        MIN("Answer"."createdAt") + INTERVAL '24 hours'
    FROM 
        "Answer"
    WHERE
        "createdAt" >= '2023-12-03'::DATE - INTERVAL '5.5 hours'
        AND "createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ
    GROUP BY
        "userId"
    ON CONFLICT ("scheduledAt", "userId", "contextId", "contextType") 
    WHERE "id" > 312384763 
    AND "scheduledAt" IS NOT NULL DO NOTHING;
END;
$$;

ALTER FUNCTION insert_notification_from_answer(TEXT, TEXT) OWNER TO learner;
