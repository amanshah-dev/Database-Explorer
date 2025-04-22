create function "PopulateDailyUserEvent"(days integer DEFAULT 1, startdate date DEFAULT '2020-01-01'::date) returns void
    language plpgsql
as
$$
DECLARE
    maxDate DATE;
BEGIN
    SELECT max("eventDate") FROM "DailyUserEvent" WHERE "courseId" IS NULL INTO maxDate;

    IF maxDate IS NOT NULL AND startDate = '2020-01-01' THEN
        startDate := maxDate + 1;
    END IF;

    IF NOW() AT TIME ZONE 'Asia/Kolkata' < startDate THEN
        startDate := (NOW() AT TIME ZONE 'Asia/Kolkata')::DATE;
    END IF;

    INSERT INTO "DailyUserEvent" ("userId", "event", "eventDate", "eventCount", "eventInfo")
    SELECT
        "userId",
        'Question',
        startDate,
        COUNT(DISTINCT "questionId"),
        jsonb_build_object('firstAnswerTime', MIN("createdAt"))
    FROM
        "Answer"
    WHERE
        "createdAt" >= startDate - INTERVAL '5.5 hours' AND "createdAt" < startDate + days - INTERVAL '5.5 hours'
        AND "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ
    GROUP BY
        "userId"
    ON CONFLICT ("userId", "event", "eventDate") WHERE "courseId" IS NULL
    DO UPDATE
    SET
        "eventCount" = EXCLUDED."eventCount",
        "eventInfo" = EXCLUDED."eventInfo";
END;
$$;

alter function "PopulateDailyUserEvent"(integer, date) owner to learner;
