create function "PruneDailyUserEvent"(lastdate date DEFAULT (CURRENT_DATE - '3 mons'::interval)) returns void
    language plpgsql
as
$$
BEGIN
  if lastDate > (current_date - interval '3 month') then
    RAISE EXCEPTION 'last date % for pruning is more than date 3 months ago %. This is a safety mechanism. Currently, we need 1 month data atleast.', lastDate, current_date - interval '3 month';
  end if;

  insert into "DailyUserEvent" ("userId", "eventDate", "event", "eventCount", "courseId")
  select "userId", lastDate, "event", sum("eventCount"), null
  from "DailyUserEvent"
  where "eventDate" <= lastDate and "courseId" is null
  group by "userId", "event"
  on conflict ("userId", "event", "eventDate") WHERE "courseId" IS NULL 
  DO UPDATE SET "eventCount" = EXCLUDED."eventCount";

  delete from "DailyUserEvent" where "courseId" is null and "eventDate" < lastDate;
END
$$;

alter function "PruneDailyUserEvent"(date) owner to learner;
