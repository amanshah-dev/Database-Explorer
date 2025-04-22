create function "PopulateDailyUserEventMeta"(days integer DEFAULT 1, startdate date DEFAULT '2020-01-01'::date) returns void
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

        insert into 
            "DailyUserEvent" ("userId", "event", "eventDate", "eventCount") 
        select user_id, column1, start_date,
            case column1 
                when 'NcertQuestion' then ncert_questions
                when 'TestQuestion' then test_questions
                when 'PhysicsQuestion' then physics_questions
                when 'ChemistryQuestion' then chemistry_questions
                when 'PhysicsCorrectQuestion' then physics_correct_questions
                when 'ChemistryCorrectQuestion' then chemistry_correct_questions
                when 'CorrectQuestion' then correct_questions
                else null end as event_count
        from (
                    select 
                        t."userId" as user_id,
                        startDate as start_date,
                        count(distinct(t."questionId")) FILTER (where t."ncert" = true) AS ncert_questions,
                        count(distinct(t."questionId")) FILTER (where t."testAttemptId" is not null) AS test_questions,
                        count(distinct(t."questionId")) FILTER (where t.subject_name = 'Physics') AS physics_questions,
                        count(distinct(t."questionId")) FILTER (where t.subject_name = 'Chemistry') AS chemistry_questions,
                        count(distinct(t."questionId")) FILTER (where t.isCorrect = true) AS correct_questions,
                        count(distinct(t."questionId")) FILTER (where t.subject_name = 'Physics' and t.isCorrect = true) AS physics_correct_questions,
                        count(distinct(t."questionId")) FILTER (where t.subject_name = 'Chemistry' and t.isCorrect = true) AS chemistry_correct_questions
                    from 
                        (select 
                                *,
                                case
                                    when "Question"."subjectId" = 54 or "Question"."subjectId" = 1214 then 'Chemistry'
                                    when "Question"."subjectId" = 55 or "Question"."subjectId" = 1215 then 'Physics'
                                else
                                    null
                                end as subject_name,
                                case 
                                    when "correctOptionIndex" = "userAnswer" then true
                                else 
                                    false 
                                end as isCorrect
                            from "Answer" 
                            inner join "Question" on 
                                "Question"."id" = "Answer"."questionId" and "Question"."deleted" = false
                            where 
                                "Answer"."createdAt" >= startDate - interval '5.5 hours' and "Answer"."createdAt" < startDate + days - interval '5.5 hours'
                        ) t
                        group by t."userId"
                    ) t1, (values ('NcertQuestion'), ('TestQuestion'), ('PhysicsQuestion'), ('ChemistryQuestion'), ('PhysicsCorrectQuestion'), ('ChemistryCorrectQuestion'), ('CorrectQuestion')) t2
        on conflict ("userId", "event", "eventDate") where "courseId" is null 
        do update set "eventCount" = EXCLUDED."eventCount";
    END
$$;

alter function "PopulateDailyUserEventMeta"(integer, date) owner to learner;
