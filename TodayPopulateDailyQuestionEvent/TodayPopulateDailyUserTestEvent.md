create function "TodayPopulateDailyUserTestEvent"() returns void
    language plpgsql
as
$$
     DECLARE
          today DATE;
          maxDate DATE;
      BEGIN
        SELECT (now() - interval '5.5 hour')::date into today;
        SELECT max("eventDate") from "DailyUserTestEvent" into maxDate;
        
        -- Case when today's data is already populated
        if maxDate = today then
            raise notice 'populate today''s data only';
            PERFORM "PopulateDailyUserTestQuestionEvent"('2123625, 2123626, 2123627, 2123628, 2123629, 2123630, 2123631, 2123632, 2123633, 2123634, 2123635, 2123636, 2123637, 2123638, 2123639, 2123640, 2123641, 2123642, 2123643', 1, today);
        end if;
        
        -- Case when yesterday's data and today's data need to be populated
        if (maxDate + interval '1 day')::date = today then
            raise notice 'populate full yesterday''s data';
            PERFORM "PopulateDailyUserTestQuestionEvent"('2123625, 2123626, 2123627, 2123628, 2123629, 2123630, 2123631, 2123632, 2123633, 2123634, 2123635, 2123636, 2123637, 2123638, 2123639, 2123640, 2123641, 2123642, 2123643', 1, maxDate);
            raise notice 'populate today''s data';
            PERFORM "PopulateDailyUserTestQuestionEvent"('2123625, 2123626, 2123627, 2123628, 2123629, 2123630, 2123631, 2123632, 2123633, 2123634, 2123635, 2123636, 2123637, 2123638, 2123639, 2123640, 2123641, 2123642, 2123643', 1, today);
        end if;
        
        -- Case when data for missing days needs to be populated
        if (maxDate + interval '1 day')::date < today then
            raise notice 'populate %''s data', maxDate + 1;
            PERFORM "PopulateDailyUserTestQuestionEvent"('2123625, 2123626, 2123627, 2123628, 2123629, 2123630, 2123631, 2123632, 2123633, 2123634, 2123635, 2123636, 2123637, 2123638, 2123639, 2123640, 2123641, 2123642, 2123643', 1, maxDate + 1);
        end if;
      END
$$;

alter function "TodayPopulateDailyUserTestEvent"() owner to learner;
