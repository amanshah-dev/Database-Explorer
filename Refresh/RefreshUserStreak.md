create function "RefreshUserStreak"(user_id integer) returns void
    language plpgsql
as
$$
      DECLARE
        dateLower TIMESTAMP;
        dateUpper TIMESTAMP;
      BEGIN
        SELECT CURRENT_DATE::TIMESTAMP AT TIME ZONE 'Asia/Kolkata' - interval '2 day' into dateLower;
        SELECT CURRENT_DATE::TIMESTAMP AT TIME ZONE 'Asia/Kolkata' + interval '1 day' into dateUpper;

        insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount") 
          select 
            "userId", 'Question', ("createdAt" at time zone 'Asia/Kolkata')::DATE as "eventDate", 
            count(distinct("questionId")) 
          from 
            "Answer" 
          where 
            "createdAt" >= dateLower and 
            "createdAt" < dateUpper and 
            "userId" = user_id
            and "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ -- FORCE USE OF PARTIAL INDEX ON ANSWER 
          group by 
            "userId", "eventDate" 
          on conflict ("userId", "event", "eventDate") 
            where "courseId" is null 
          do update set "eventCount" = EXCLUDED."eventCount";

        insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount") 
          select 
            "userId", 'TestQuestion', ("createdAt" at time zone 'Asia/Kolkata')::DATE as "eventDate", 
            count(distinct(case when "testAttemptId" is not null then "questionId" else null end)) 
          from 
            "Answer" 
          where 
            "createdAt" >= dateLower and 
            "createdAt" < dateUpper  and 
            "userId" = user_id 
            and "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ -- FORCE USE OF PARTIAL INDEX ON ANSWER
          group by 
            "userId", "eventDate" 
          on conflict ("userId", "event", "eventDate") 
          where "courseId" is null 
          do update set "eventCount" = EXCLUDED."eventCount";

        insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount")
          select 
            "userId", 'NcertQuestion', ("createdAt" at time zone 'Asia/Kolkata')::DATE as "eventDate", 
            count(distinct(select "id" from "Question" where "id" = "questionId" and "ncert" = true)) 
          from 
            "Answer" 
          where 
            "createdAt" >= dateLower and 
            "createdAt" < dateUpper  and 
            "userId" = user_id 
            and "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ -- FORCE USE OF PARTIAL INDEX ON ANSWER
          group by 
            "userId", "eventDate" 
          on conflict ("userId", "event", "eventDate") 
          where "courseId" is null 
          do update set "eventCount" = EXCLUDED."eventCount";

        insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount") 
          select 
            "userId", 'CorrectQuestion', ("createdAt" at time zone 'Asia/Kolkata')::DATE as "eventDate", 
            count(distinct(select "id" from "Question" where "id" = "questionId" and "correctOptionIndex" = "userAnswer")) 
          from 
            "Answer" 
          where 
            "createdAt" >= dateLower and
            "createdAt" < dateUpper  and 
            "userId" = user_id 
            and "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ -- FORCE USE OF PARTIAL INDEX ON ANSWER
          group by 
            "userId", "eventDate" 
          on conflict ("userId", "event", "eventDate") 
          where "courseId" is null 
          do update set "eventCount" = EXCLUDED."eventCount";

        insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount") 
          select 
            "userId", 'PhysicsQuestion', ("createdAt" at time zone 'Asia/Kolkata')::DATE as "eventDate", 
            count(distinct(select "id" from "Question" where "id" = "questionId" and "subjectId" in (55, 1215))) 
          from 
            "Answer" 
          where 
            "createdAt" >= dateLower and 
            "createdAt" < dateUpper and 
            "userId" = user_id 
            and "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ -- FORCE USE OF PARTIAL INDEX ON ANSWER
            group by 
              "userId", "eventDate" 
            on conflict ("userId", "event", "eventDate") 
            where "courseId" is null 
            do update set "eventCount" = EXCLUDED."eventCount";

        insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount") 
          select 
            "userId", 'ChemistryQuestion', ("createdAt" at time zone 'Asia/Kolkata')::DATE as "eventDate", 
            count(distinct(select "id" from "Question" where "id" = "questionId" and "subjectId" in (54, 1214))) 
          from 
            "Answer" 
          where 
            "createdAt" >= dateLower and 
            "createdAt" < dateUpper and 
            "userId" = user_id 
            and "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ -- FORCE USE OF PARTIAL INDEX ON ANSWER
          group by 
            "userId", "eventDate" 
          on conflict ("userId", "event", "eventDate") 
          where "courseId" is null 
          do update set "eventCount" = EXCLUDED."eventCount";

        insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount") 
          select 
            "userId", 'PhysicsCorrectQuestion', ("createdAt" at time zone 'Asia/Kolkata')::DATE as "eventDate", 
            count(distinct(select "id" from "Question" where "id" = "questionId" and "subjectId" in (55, 1215) and "correctOptionIndex" = "userAnswer")) 
          from 
            "Answer" 
          where 
            "createdAt" >= dateLower and 
            "createdAt" < dateUpper  and 
            "userId" = user_id 
            and "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ -- FORCE USE OF PARTIAL INDEX ON ANSWER
          group by 
            "userId", "eventDate" 
          on conflict ("userId", "event", "eventDate") 
          where 
            "courseId" is null 
          do update set "eventCount" = EXCLUDED."eventCount";

        insert into "DailyUserEvent" ("userId", "event", "eventDate", "eventCount") 
          select 
            "userId", 'ChemistryCorrectQuestion', ("createdAt" at time zone 'Asia/Kolkata')::DATE as "eventDate", 
            count(distinct(select "id" from "Question" where "id" = "questionId" and "subjectId" in (54, 1214) and "correctOptionIndex" = "userAnswer")) 
          from 
            "Answer" 
          where 
            "createdAt" >= dateLower and 
            "createdAt" < dateUpper  and 
            "userId" = user_id 
            and "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ -- FORCE USE OF PARTIAL INDEX ON ANSWER
          group by 
            "userId", "eventDate" 
          on conflict ("userId", "event", "eventDate") 
          where 
            "courseId" is null 
          do update set "eventCount" = EXCLUDED."eventCount";
      END
$$;

alter function "RefreshUserStreak"(integer) owner to learner;
