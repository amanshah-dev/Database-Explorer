-- Create function to check if the user has achieved the required best score in a test
create function checktestbestscore(userid integer, requiredmarks integer) returns boolean
    language plpgsql
as
$$
DECLARE
    marks_achieved INT;  -- Variable to store the highest marks achieved by the user
BEGIN
    -- Query to get the maximum total marks achieved by the user in tests with exactly 200 questions
    SELECT MAX((ta.result ->> 'totalMarks')::INT)
    INTO marks_achieved
    FROM "TestAttempt" ta
             JOIN "Test" t ON ta."testId" = t."id" AND t."numQuestions" = 200  -- Ensure only tests with 200 questions are considered
    WHERE ta."userId" = userid;

    -- Log the highest marks achieved by the user for debugging purposes
    RAISE NOTICE 'Highest marks achieved by user %: %', userid, marks_achieved;

    -- Return true if the highest marks achieved are greater than or equal to the required marks, otherwise false
    RETURN marks_achieved >= requiredmarks;
END;
$$;

-- Alter function owner
alter function checktestbestscore(integer, integer) owner to neetprep_rw;
