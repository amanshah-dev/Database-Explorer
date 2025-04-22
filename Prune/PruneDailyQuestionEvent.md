create function "PruneDailyQuestionEvent"(lastdate date DEFAULT (CURRENT_DATE - '2 mons'::interval)) returns void
    language plpgsql
as
$$
BEGIN
  if lastDate > (current_date - interval '2 month') then
    RAISE EXCEPTION 'last date % for pruning is more than date 2 months ago %. This is a safety mechanism. Currently, we need 1 month data just to be safe while checking for any issue later on.', lastDate, current_date - interval '2 month';
  end if;

  insert into "DailyQuestionEvent" ("questionId", "eventDate", "eventType", "eventCount")
  select "questionId", lastDate, "eventType", sum("eventCount")
  from "DailyQuestionEvent"
  where "eventDate" <= lastDate
  group by "questionId", "eventType"
  on conflict ("questionId", "eventType", "eventDate") 
  DO UPDATE SET "eventCount" = EXCLUDED."eventCount";

  delete from "DailyQuestionEvent" where "eventDate" < lastDate;
END
$$;

alter function "PruneDailyQuestionEvent"(date) owner to learner;
