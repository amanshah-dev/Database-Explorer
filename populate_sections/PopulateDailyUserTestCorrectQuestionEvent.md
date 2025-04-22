create function "PopulateDailyUserTestCorrectQuestionEvent"(testid integer, days integer DEFAULT 1, startdate date DEFAULT '2023-01-01'::date) returns void
    language plpgsql
as
$$
DECLARE
    maxDate DATE;
BEGIN
    SELECT max("eventDate") from "DailyUserTestEvent" into maxDate;
    
    if maxDate is not null and startDate = '2023-01-01' then
        startDate := maxDate + 1;
    end if;
    
    if now() at time zone 'Asia/Kolkata' < startDate then
        startDate := (now() at time zone 'Asia/Kolkata')::DATE;
    end if;

    insert into "DailyUserTestEvent" ("userId", "testId", "event", "eventDate", "eventCount")
    select "userId", "testId", 'CorrectQuestion', startDate, 
           count(distinct("Answer"."questionId"))
    from "Answer", "TestQuestion", "Question"
    where "Answer"."createdAt" >= startDate - interval '5.5 hours' 
      and "Answer"."createdAt" < startDate + days - interval '5.5 hours' 
      and "TestQuestion"."testId" = testid 
      and "TestQuestion"."questionId" = "Answer"."questionId"
      and "Question"."id" = "TestQuestion"."questionId" 
      and "Answer"."userAnswer" = "Question"."correctOptionIndex"
    group by "userId", "testId"
    on conflict ("userId", "testId", "event", "eventDate") 
    do update set "eventCount" = EXCLUDED."eventCount";
END
$$;

alter function "PopulateDailyUserTestCorrectQuestionEvent"(integer, integer, date) owner to learner;
