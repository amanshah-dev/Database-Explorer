create function "PopulateDailyUserTestSubmitEvent"(days integer DEFAULT 1, startdate date DEFAULT '2023-01-01'::date) returns void
    language plpgsql
as
$$
DECLARE
    maxDate DATE;
BEGIN
    SELECT max("eventDate") from "DailyUserEvent" where "courseId" is null and "event" = 'TestSubmit' into maxDate;
    
    if maxDate is not null and startDate = '2023-01-01' then
        startDate := maxDate + 1;
    end if;

    if now() at time zone 'Asia/Kolkata' < startDate then
        startDate := (now() at time zone 'Asia/Kolkata')::DATE;
    end if;

    insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount")
    select "userId", 'TestSubmit', startDate, count(distinct("testId"))
    from "TestAttempt"
    where "TestAttempt"."createdAt" >= startDate - interval '5.5 hours' 
      and "TestAttempt"."createdAt" < startDate + days - interval '5.5 hours'
      and "TestAttempt"."completed" = true 
      and exists (select * from "User" where "User"."id" = "TestAttempt"."userId")
    group by "userId"
    on conflict ("userId", "event", "eventDate") where "courseId" is null 
    do update set "eventCount" = EXCLUDED."eventCount";
END
$$;

alter function "PopulateDailyUserTestSubmitEvent"(integer, date) owner to learner;
