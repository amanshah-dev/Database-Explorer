create function "PruneDailyUserTestEvent"(lastdate date DEFAULT (CURRENT_DATE - '7 days'::interval)) returns void
    language plpgsql
as
$$
BEGIN
  if lastDate > (current_date - interval '7 days') then
    RAISE EXCEPTION 'last date % for pruning is more than date 7 days ago %. This is a safety mechanism. Currently, we need atleast a few days data for some debugging requirements. Later, we can reduce minimum number of days data.', lastDate, current_date - interval '7 days';
  end if;

  insert into "DailyUserTestEvent" ("userId", "eventDate", "event", "eventCount", "testId")
  select "userId", lastDate, "event", sum("eventCount"), "testId"
  from "DailyUserTestEvent"
  where "eventDate" <= lastDate
  group by "userId", "event", "testId"
  on conflict ("userId", "event", "eventDate", "testId") 
  DO UPDATE SET "eventCount" = EXCLUDED."eventCount";

  delete from "DailyUserTestEvent" where "eventDate" < lastDate;
END
$$;

alter function "PruneDailyUserTestEvent"(date) owner to learner;
