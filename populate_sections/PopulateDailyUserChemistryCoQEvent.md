create function "PopulateDailyUserChemistryCoQEvent"(days integer DEFAULT 1, startdate date DEFAULT '2020-01-01'::date) returns void
    language plpgsql
as
$$
DECLARE
    maxDate DATE;
BEGIN
    SELECT max("eventDate") from "DailyUserEvent" where "courseId" is null into maxDate;
    
    if maxDate is not null and startDate = '2020-01-01' then
        startDate := maxDate + 1;
    end if;
    
    if now() at time zone 'Asia/Kolkata' < startDate then
        startDate := (now() at time zone 'Asia/Kolkata')::DATE;
    end if;
    
    insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount") 
    select "userId", 'ChemistryCorrectQuestion', startDate, 
           count(distinct(select "id" from "Question" where "id" = "questionId" and "subjectId" in (54, 1214))) 
    from "Answer"
    where "Answer"."createdAt" >= startDate - interval '5.5 hours' 
      and "Answer"."createdAt" < startDate + days - interval '5.5 hours'
    group by "userId"
    on conflict ("userId", "event", "eventDate") where "courseId" is null 
    do update set "eventCount" = EXCLUDED."eventCount";
END;
$$;

alter function "PopulateDailyUserChemistryCoQEvent"(integer, date) owner to learner;
