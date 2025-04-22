-- Create function to check if a user has a sufficient streak of correct Chemistry questions
create function checkchemistrystreak(userid integer, questions integer) returns boolean
    language plpgsql
as
$$
DECLARE
    question_count INT;  -- Variable to store the count of correct Chemistry questions
BEGIN
    -- Count the number of correct Chemistry questions for the given user
    SELECT COUNT(*)
    INTO question_count
    FROM "DailyUserEvent"
    WHERE "userId" = userId
      AND "event" = 'ChemistryCorrectQuestion';

    -- Return true if the user has at least the required number of correct Chemistry questions
    RETURN question_count >= questions;
EXCEPTION
    WHEN OTHERS THEN
        -- Catch any errors and log the error message
        RAISE NOTICE 'Failed to check Chemistry streak for user % with error: %', userId, SQLERRM;
        RETURN FALSE;  -- Return FALSE in case of an error
END;
$$;

-- Alter function owner
alter function checkchemistrystreak(integer, integer) owner to neetprep_rw;
