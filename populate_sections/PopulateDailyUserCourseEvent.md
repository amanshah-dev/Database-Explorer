create function "PopulateDailyUserCourseEvent"(days integer DEFAULT 1, courseid integer DEFAULT 255) returns void
    language plpgsql
as
$$
DECLARE
    maxDate DATE;
    startDate DATE;
BEGIN
    select max("eventDate") from "DailyUserEvent" where "courseId" = courseId into maxDate;
    
    if maxDate is not null then
        startDate := maxDate + 1;
    else
        startDate := '2020-01-01';
    end if;
    
    insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount", "courseId") 
    select "userId", 'Question', startDate, count(distinct("questionId")), courseId 
    from "Answer", "QuestionCourse"
    where "createdAt" at time zone 'utc' at time zone 'Asia/Kolkata' >= startDate 
      and "createdAt" at time zone 'utc' at time zone 'Asia/Kolkata' < startDate + days
      and "QuestionCourse"."courseId" = courseId
      and "QuestionCourse"."questionId" = "Answer"."questionId"
    group by "userId";
END
$$;

alter function "PopulateDailyUserCourseEvent"(integer, integer) owner to learner;
