-- Create function to check if a user has a sufficient streak of correct Physics questions
create function checkphysicsstreak(userid integer, questions integer) returns boolean
    language plpgsql
as
$$
DECLARE
    question_count INT;  -- Variable to store the count of correct Physics questions
BEGIN
    -- Count the number of correct Physics questions for the given user
    SELECT COUNT(*)
    INTO question_count
    FROM "DailyUserEvent"
    WHERE "userId" = userId
      AND "event" = 'PhysicsCorrectQuestion';

    -- Return true if the user has at least the required number of correct Physics questions
    RETURN question_count >= questions;
END;
$$;

-- Alter function owner
alter function checkphysicsstreak(integer, integer) owner to neetprep_rw;
