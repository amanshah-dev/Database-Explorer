-- Create function to check if a user has a sufficient streak of practice sessions
create function checkpracticestreak(userid integer, questions integer) returns boolean
    language plpgsql
as
$$
DECLARE
    practice_count INT;  -- Variable to store the count of practice sessions
BEGIN
    -- Count the number of practice sessions for the given user from the UserDpp table
    SELECT COUNT(*)
    INTO practice_count
    FROM "UserDpp"
    WHERE "userId" = userId
      AND "dppType" = 'practice';

    -- Log the practice count for debugging purposes
    RAISE NOTICE 'Practice count for user %: %', userId, practice_count;

    -- Return true if the user has at least the required number of practice sessions
    RETURN practice_count >= questions;
END;
$$;

-- Alter function owner
alter function checkpracticestreak(integer, integer) owner to neetprep_rw;
