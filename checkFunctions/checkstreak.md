-- Create function to check if users have a sufficient streak of events within a specified number of days
create function checkstreak(userids bigint[], days integer) returns boolean
    language plpgsql
as
$$
DECLARE
    streak_start DATE;    -- Variable to store the start date of the streak
    streak_end   DATE;    -- Variable to store the end date of the streak
    event_count  INT;     -- Variable to store the count of distinct event dates
BEGIN
    -- Set the end date as the current date and calculate the start date based on the provided streak length
    streak_end := CURRENT_DATE;
    streak_start := streak_end - (days - 1);

    -- Query the DailyUserEvent table to count distinct event dates for the user(s) within the streak period
    SELECT COUNT(DISTINCT "eventDate")
    INTO event_count
    FROM "DailyUserEvent"
    WHERE "userId" = any (userids)            -- Check for the provided user IDs
      AND "eventDate" BETWEEN streak_start AND streak_end
      AND "event" LIKE '%Question%'           -- Filter for events related to questions
      AND "eventCount" > 0;                   -- Ensure that the event count is greater than 0

    -- Check if the number of distinct dates with events matches the required number of days
    RETURN event_count = days;
END;
$$;

-- Alter function owner
alter function checkstreak(bigint[], integer) owner to neetprep_rw;
