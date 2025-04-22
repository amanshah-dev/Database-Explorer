create function "PopulateDailyUserEventForUser"(days integer, userid integer, startdate date DEFAULT '2020-01-01'::date) returns void
    language plpgsql
as
$$
DECLARE
    maxDate DATE;
BEGIN
    INSERT INTO "DailyUserEvent" ("userId", "event", "eventDate", "eventCount", "eventInfo")
    SELECT
        "userId",
        'Question',
        ("createdAt" + INTERVAL '5.5 hours')::date as "eventDate",
        COUNT(DISTINCT "questionId"),
        jsonb_build_object('firstAnswerTime', MIN("createdAt"))
    FROM
        "Answer"
    WHERE
        "createdAt" >= startDate::date - INTERVAL '5.5 hours' AND "createdAt" < startDate::date + days - INTERVAL '5.5 hours'
        AND "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ
        AND "userId" = userid
    GROUP BY
        "userId", "eventDate"
    ON CONFLICT ("userId", "event", "eventDate") WHERE "courseId" IS NULL
    DO UPDATE
    SET
        "eventCount" = EXCLUDED."eventCount",
        "eventInfo" = EXCLUDED."eventInfo";
END;
$$;

alter function "PopulateDailyUserEventForUser"(integer, integer, date) owner to neetprep_rw;
