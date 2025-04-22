create function "PopulateDailyUserCourseEvent"(days integer DEFAULT 1, courseid integer DEFAULT 255, startdate date DEFAULT '2020-01-01'::date) returns void
    language plpgsql
as
$$
DECLARE
    maxDate DATE;
BEGIN
    select max("eventDate") from "DailyUserEvent" where "courseId" = courseId into maxDate;
    
    if maxDate is not null and startDate = '2020-01-01' then
        startDate := maxDate + 1;
    end if;
    
    if now() at time zone 'Asia/Kolkata' < startDate then
        startDate := (now() at time zone 'Asia/Kolkata')::DATE;
    end if;
    
    insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount", "courseId") 
    select "userId", 'Question', startDate, count(distinct("questionId")), courseid 
    from "Answer", "QuestionCourse"
    where "createdAt" >= startDate - interval '5.5 hours' 
      and "createdAt" < startDate + days - interval '5.5 hours'
      and "QuestionCourse"."courseId" = courseid
      and "QuestionCourse"."questionId" = "Answer"."questionId"
    group by "userId"
    on conflict ("userId", "event", "eventDate", "courseId") 
    do update set "eventCount" = EXCLUDED."eventCount";
END
$$;

alter function "PopulateDailyUserCourseEvent"(integer, integer, date) owner to learner;
