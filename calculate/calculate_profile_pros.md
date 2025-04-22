-- Create function to calculate profile pros based on filled fields
create function calculate_profile_pros(userids bigint[], level integer) returns integer[]
    language plpgsql
as
$$
DECLARE
    fieldsToCheck TEXT[] := ARRAY ['displayName', 'phone', 'email', 'dob', 'picture', 'gender', 'neetExamYear',
        'address', 'state', 'pincode', 'city', 'board', 'userClass', 'goal'];

    -- Declare variable to count filled fields
    profilePros   INT[]  := ARRAY []::BIGINT[];
    total_fields  INT    := 14; -- Total number of fields to check
    l             INT;
BEGIN
    total_fields := array_length(fieldsToCheck, 1);
    l := CEIL(level * total_fields / 100.0); -- Calculate the number of fields that need to be filled based on the level

    -- Fetch the user profile data using the count_non_null_fields function
    SELECT ARRAY_AGG(cnnf.userId)
    INTO profilePros
    FROM count_non_null_fields(userIds, fieldsToCheck) cnnf
    WHERE cnnf.non_null_count >= l; -- Ensure the number of non-null fields meets the requirement

    -- Return the list of user IDs that meet the profile completion requirement
    RETURN profilePros;
END;
$$;

-- Alter function owner
alter function calculate_profile_pros(bigint[], integer) owner to neetprep_rw;
