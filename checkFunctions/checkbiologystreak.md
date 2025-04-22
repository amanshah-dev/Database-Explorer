-- Create function to check if a user has a sufficient streak of correct Biology questions
create function checkbiologystreak(userid integer, questions integer) returns boolean
    language plpgsql
as
$$
DECLARE
    biology_questions       INT;
    chemistry_questions     INT;
    physics_questions       INT;
    total_correct_questions INT;
BEGIN
    -- Count the total correct questions
    SELECT COUNT(*)
    INTO total_correct_questions
    FROM "DailyUserEvent"
    WHERE "userId" = userId
      AND "event" = 'CorrectQuestion';

    -- Count the correct Chemistry questions
    SELECT COUNT(*)
    INTO chemistry_questions
    FROM "DailyUserEvent"
    WHERE "userId" = userId
      AND "event" = 'ChemistryCorrectQuestion';

    -- Count the correct Physics questions
    SELECT COUNT(*)
    INTO physics_questions
    FROM "DailyUserEvent"
    WHERE "userId" = userId
      AND "event" = 'PhysicsCorrectQuestion';

    -- Calculating Biology questions by subtracting Chemistry and Physics counts from total correct questions
    biology_questions := total_correct_questions - (chemistry_questions + physics_questions);

    -- Log the counts for debugging purposes
    RAISE NOTICE 'Total Correct Questions: %, Chemistry Questions: %, Physics Questions: %, Biology Questions: %', total_correct_questions, chemistry_questions, physics_questions, biology_questions;

    -- Check if the Biology questions meet the required number
    RETURN biology_questions >= questions;
EXCEPTION
    WHEN OTHERS THEN
        -- Catch any errors and log the error message
        RAISE NOTICE 'Failed to check Biology streak for user % with error: %', userId, SQLERRM;
        RETURN FALSE;
END;
$$;

-- Alter function owner
alter function checkbiologystreak(integer, integer) owner to neetprep_rw;
