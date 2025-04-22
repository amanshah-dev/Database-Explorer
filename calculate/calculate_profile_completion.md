-- Create function to calculate profile completion based on filled fields
create function calculate_profile_completion(userid integer, level integer) returns boolean
    language plpgsql
as
$$
DECLARE
    -- Declare variables to store profile data
    displayName           TEXT;
    mobileNumber          TEXT;
    email                 TEXT;
    dob                   DATE;
    profileImage          TEXT;
    gender                TEXT;
    neetExamYear          INTEGER;
    address               TEXT;
    state                 TEXT;
    district              TEXT;
    pinCode               TEXT;
    educationBoard        TEXT;
    userClass             TEXT;
    goal                  TEXT;

    -- Declare variable to count filled fields
    filled_fields_count   INT := 0;
    total_fields          INT := 14; -- Total number of fields to check
    completion_percentage NUMERIC;
BEGIN
    -- Fetch the user profile data
    SELECT up."displayName",
           up."phone"     AS mobileNumber,
           up."email",
           up."dob",
           up."picture"   AS profileImage,
           up."gender",
           up."neetExamYear",
           up."address",
           up."state",
           up."pincode",
           up."city"      AS district,
           up."board"     AS educationBoard,
           up."userClass" AS userClass,
           up."goal"
    INTO displayName, mobileNumber, email, dob, profileImage, gender, neetExamYear, address, state, pinCode, district, educationBoard, userClass, goal
    FROM "UserProfile" up
    WHERE up."userId" = userId;

    -- Check each field and increment the filled fields count if not empty or null
    IF displayName IS NOT NULL AND displayName <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF mobileNumber IS NOT NULL AND mobileNumber <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF email IS NOT NULL AND email <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF dob IS NOT NULL THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF profileImage IS NOT NULL AND profileImage <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF gender IS NOT NULL AND gender <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF neetExamYear IS NOT NULL THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF address IS NOT NULL AND address <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF state IS NOT NULL AND state <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF pinCode IS NOT NULL AND pinCode <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF district IS NOT NULL AND district <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF educationBoard IS NOT NULL AND educationBoard <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF userClass IS NOT NULL AND userClass <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    IF goal IS NOT NULL AND goal <> '' THEN
        filled_fields_count := filled_fields_count + 1;
    END IF;

    -- Calculate the completion percentage
    completion_percentage := (filled_fields_count::NUMERIC / total_fields) * 100;

    -- Return whether the completion percentage is above or equal to the level
    RETURN completion_percentage >= level;
END;
$$;

-- Alter function owner
alter function calculate_profile_completion(integer, integer) owner to neetprep_rw;
