create function "RankUpdateOfflineTestAttempts"(testid integer) returns void
    language plpgsql
as
$$
BEGIN
  update "OfflineTestAttempt" set "rank" = t1."rank" from
    (select "score", "attemptId", "mode", rank() over (ORDER BY "score" DESC) as "rank" 
     from (
         select ("result"->>'totalMarks')::integer as "score", "TestAttempt"."id" as "attemptId", 'Online' as "mode"
         from "TestAttempt", "Test"
         where (("TestAttempt"."testId" = testId) 
                or ("TestAttempt"."testId" in (select "origId" from "DuplicateTest" where "dupId" = testId)))
           and "TestAttempt"."testId" = "Test"."id"
           and "TestAttempt"."completed" = true
           and "TestAttempt"."finishedAt" < "Test"."reviewAt"
           and "result" is not null
           and "TestAttempt"."userId" not in (
               select "userId" 
               from "TestAttempt" ta 
               where ta."testId" = (select "origId" from "DuplicateTest" where "dupId" = testid) 
                 and ta."offlineTestAttemptId" is not null
           )
        union
         select COALESCE((select (ta."result"->>'totalMarks')::integer 
                          from "TestAttempt" ta 
                          where ta."offlineTestAttemptId" = "OfflineTestAttempt"."id" 
                          limit 1), 
                         ("result"->>'totalMarks')::integer) as "score", 
                         "OfflineTestAttempt"."id" as "attemptId", 'Offline' as "mode"
         from "OfflineTestAttempt"
         where (("OfflineTestAttempt"."rawTestId" = testId) 
                or ("OfflineTestAttempt"."rawTestId" in (
                    select "dupId" from "DuplicateTest" 
                    where "origId" = (select "origId" from "DuplicateTest" dt where dt."dupId" = testId)
                )))
           and "result" is not null
    ) t
    order by "score" desc
    ) t1 
    where t1."attemptId" = "OfflineTestAttempt"."id";
END
$$;

alter function "RankUpdateOfflineTestAttempts"(integer) owner to learner;
