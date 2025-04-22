create function "PopulateDailyUserVideoViewEvent"(days integer DEFAULT 1, startdate date DEFAULT '2023-01-19'::date) returns void
    language plpgsql
as
$$
DECLARE
    maxDate DATE;
BEGIN
    SELECT max("eventDate") from "DailyUserEvent" where "courseId" is null and "event" = 'VideoView' into maxDate;
    
    if maxDate is not null and startDate = '2023-01-19' then
        startDate := maxDate + 1;
    end if;

    if now() at time zone 'Asia/Kolkata' < startDate then
        startDate := (now() at time zone 'Asia/Kolkata')::DATE;
    end if;

    insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount")
    select "userId", 'VideoView', startDate, count(distinct("videoId"))
    from "UserVideoStat"
    where "UserVideoStat"."createdAt" >= startDate - interval '5.5 hours' 
      and "UserVideoStat"."createdAt" < startDate + days - interval '5.5 hours'
      and "UserVideoStat"."completed" = true 
      and exists (select * from "User" where "User"."id" = "UserVideoStat"."userId")
    group by "userId"
    on conflict ("userId", "event", "eventDate") where "courseId" is null 
    do update set "eventCount" = EXCLUDED."eventCount";
END
$$;

alter function "PopulateDailyUserVideoViewEvent"(integer, date) owner to learner;
