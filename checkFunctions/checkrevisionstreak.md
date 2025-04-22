-- Create function to check if a user has a sufficient streak of revision sessions
create function checkrevisionstreak(userid integer, questions integer) returns boolean
    language plpgsql
as
$$
DECLARE
    revision_count INT;  -- Variable to store the count of revision sessions
BEGIN
    -- Count the number of revision sessions for the given user from the UserDpp table
    SELECT COUNT(*)
    INTO revision_count
    FROM "UserDpp"
    WHERE "userId" = userId
      AND "dppType" = 'revision';

    -- Log the revision count for debugging purposes
    RAISE NOTICE 'Revision count for user %: %', userId, revision_count;

    -- Return true if the user has at least the required number of revision sessions
    RETURN revision_count >= questions;
END;
$$;

-- Alter function owner
alter function checkrevisionstreak(integer, integer) owner to neetprep_rw;
