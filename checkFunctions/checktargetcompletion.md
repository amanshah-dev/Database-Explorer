-- Create function to check if the user has met the required number of targets
create function checktargetcompletion(userid integer, targets integer) returns boolean
    language plpgsql
as
$$
DECLARE
    target_met_count INT;  -- Variable to store the count of targets met by the user
BEGIN
    -- Query to count the number of test attempts where the user's result meets or exceeds the target score
    SELECT COUNT(*)
    INTO target_met_count
    FROM "TestAttempt"
             JOIN "Target" ON "TestAttempt"."testId" = "Target"."testId"
    WHERE "TestAttempt"."userId" = userid  -- Filter by user ID
      AND ("TestAttempt"."result" ->> 'totalMarks')::INT >= "Target"."score";  -- Ensure totalMarks >= target score

    -- Return true if the user has met the required number of targets, otherwise false
    RETURN target_met_count >= targets;
END;
$$;

-- Alter function owner
alter function checktargetcompletion(integer, integer) owner to neetprep_rw;
