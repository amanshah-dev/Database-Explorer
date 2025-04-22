CREATE FUNCTION duplicate_test_with_questions(original_test_id INTEGER, new_test_name TEXT DEFAULT ''::TEXT) RETURNS VOID
    LANGUAGE plpgsql
AS
$$
DECLARE
    new_test_id INT;
    final_test_name TEXT;
BEGIN
    -- Set the final test name based on the input
    IF new_test_name = '' THEN
        -- If the new name is empty, append " - duplicate" to the existing new name
        SELECT "name" || ' - duplicate' INTO final_test_name
        FROM "Test"
        WHERE "id" = original_test_id;
    ELSE
        -- Use the provided new name
        final_test_name := new_test_name;
    END IF;

    -- Duplicate the "Test" table entry with the final name
    INSERT INTO "Test" ("name")
    VALUES (final_test_name)
    RETURNING "id" INTO new_test_id;

    -- Duplicate the "TestQuestion" entries for the new test
    INSERT INTO "TestQuestion" ("testId", "questionId")
    SELECT new_test_id, "questionId"
    FROM "TestQuestion"
    WHERE "testId" = original_test_id;
END;
$$;

ALTER FUNCTION duplicate_test_with_questions(integer, text) OWNER TO learner;
