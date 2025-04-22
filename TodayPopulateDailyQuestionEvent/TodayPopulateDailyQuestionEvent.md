create function "TodayPopulateDailyQuestionEvent"() returns void
    language plpgsql
as
$$
     DECLARE
          today DATE;
          maxDate DATE;
      BEGIN
        SELECT (now() - interval '5.5 hour')::date into today;
        SELECT max("eventDate") from "DailyQuestionEvent" into maxDate;
        
        -- Case when today's data is already populated
        if maxDate = today then
            raise notice 'populate today''s data only';
            PERFORM "PopulateDailyQuestionEvent"(today);
        end if;
        
        -- Case when only yesterday's data is missing
        if (maxDate + interval '1 day')::date = today then
            raise notice 'populate full yesterday''s data';
            PERFORM "PopulateDailyQuestionEvent"(maxDate);
            raise notice 'populate today''s data';
            PERFORM "PopulateDailyQuestionEvent"(today);
        end if;
        
        -- Case when we need to populate data for any missing days
        if (maxDate + interval '1 day')::date < today then
            raise notice 'populate %''s data', maxDate + 1;
            PERFORM "PopulateDailyQuestionEvent"(maxDate + 1);
        end if;
      END
$$;

alter function "TodayPopulateDailyQuestionEvent"() owner to learner;
