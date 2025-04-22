create function testdategenfunc(startdate date, enddate date) returns date[]
    language plpgsql
as
$$
DECLARE
    currentDate DATE := startDate;
    result DATE[] := '{}';
    daysUntilNextSunday INT;
BEGIN
    -- Calculate days until the next Sunday from startDate
    daysUntilNextSunday := (7 - EXTRACT(DOW FROM startDate)) % 7;
    if daysUntilNextSunday = 0 then
        daysUntilNextSunday := 7; -- Ensure we start at least one week away if today is Sunday
    end if;
    currentDate := startDate + daysUntilNextSunday * INTERVAL '1 day';

    -- Check if this next Sunday is exactly 7 days away from startDate
    IF currentDate - startDate < 7 THEN
        -- Ensure the first test date is at least 7 days away from startDate
        currentDate := currentDate + INTERVAL '7 days';
    END IF;

    -- Loop to collect dates every two weeks on Sundays until endDate
    WHILE currentDate <= endDate LOOP
        result := array_append(result, currentDate);
        currentDate := currentDate + INTERVAL '14 days';  -- Advance by two weeks
    END LOOP;
    
    RETURN result;
END;
$$;

alter function testdategenfunc(date, date) owner to neetprep_rw;
