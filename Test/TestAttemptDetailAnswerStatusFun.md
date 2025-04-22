create function "TestAttemptDetailAnswerStatusFunc"(testattemptid integer) returns integer[]
    language plpgsql
as
$$
DECLARE
  result int[]; 
BEGIN
  select array(select
    /* 
    we are following a convention for returning values
    0 - not answered
    1 - answered correctly
    -1 - answered incorrectly
    */ 
   case
     when "userAnswers" ->> "Question"."id"::text is null then 0 
     when "userAnswers" ->> "Question"."id"::text = "Question"."correctOptionIndex"::text then 1
     else -1
   end
   from "TestAttempt", "TestQuestion", "Question" 
    where "TestAttempt"."id" = testAttemptId
      and "TestQuestion"."testId" = "TestAttempt"."testId"
      and "Question"."id" = "TestQuestion"."questionId"
    order by
      "TestQuestion"."seqNum" asc, "TestQuestion"."questionId" asc) into result; 
  return result;
END
$$;

alter function "TestAttemptDetailAnswerStatusFunc"(integer) owner to learner;
