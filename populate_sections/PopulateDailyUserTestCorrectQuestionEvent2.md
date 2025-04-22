create function "PopulateDailyUserTestCorrectQuestionEvent"(testids text, days integer DEFAULT 1, startdate date DEFAULT '2023-01-04'::date) returns void
    language plpgsql
as
$$
DECLARE
    maxDate DATE;
    ids INT[];
BEGIN
    ids = string_to_array(testIds, ',');

    SELECT max("eventDate") from "DailyUserTestEvent" where "event" = 'CorrectQuestion' into maxDate;
    
    if maxDate is not null and startDate = '2023-01-04' then
        startDate := maxDate + 1;
    end if;
    
    if now() at time zone 'Asia/Kolkata' < startDate then
        startDate := (now() at time zone 'Asia/Kolkata')::DATE;
    end if; 
    
    insert into "DailyUserTestEvent" ("userId", "testId", "event", "eventDate", "eventCount")
    select "userId", "testId", 'CorrectQuestion', startDate, 
           count(distinct(select "id" from "Question" where "id" = "Answer"."questionId" and "correctOptionIndex" = "userAnswer"))
    from "Answer", "TestQuestion"
    where "Answer"."testAttemptId" is null 
      and "Answer"."createdAt" >= startDate - interval '5.5 hours' 
      and "Answer"."createdAt" < startDate + days - interval '5.5 hours' 
      and "Answer"."questionId" in (select "questionId" from "TestQuestion" where "TestQuestion"."testId" = any(ids)) 
      and "TestQuestion"."questionId" = "Answer"."questionId" 
      and "TestQuestion"."testId" = any(ids) 
      and "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ
    group by "userId", "testId"
    on conflict ("userId", "testId", "event", "eventDate") 
    do update set "eventCount" = EXCLUDED."eventCount";
END
$$;

alter function "PopulateDailyUserTestCorrectQuestionEvent"(text, integer, date) owner to learner;
