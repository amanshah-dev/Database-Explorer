
CREATE FUNCTION create_temp_test_entries() RETURNS trigger
    LANGUAGE plpgsql
AS
$$
DECLARE
    temp_test_id INTEGER;
    backup_temp_test_id INTEGER;
BEGIN
    -- Insert first TempTest row
    INSERT INTO "TempTest" ("name", "testPatternId", "numQuestions", "durationInMin", "free", "sections", "positiveMarks", "negativeMarks")
    SELECT
        NEW."name",
        NEW."testPatternId",
        "numQuestions",
        "durationInMin",
        "free",
        "sections",
        "positiveMarks",
        "negativeMarks"
    FROM "TestPattern"
    WHERE "id" = NEW."testPatternId"
    RETURNING "id" INTO temp_test_id;

    -- Insert second TempTest row
    INSERT INTO "TempTest" ("name", "testPatternId", "numQuestions", "durationInMin", "free", "sections", "positiveMarks", "negativeMarks")
    SELECT
        NEW."name",
        NEW."testPatternId",
        "numQuestions",
        "durationInMin",
        "free",
        "sections",
        "positiveMarks",
        "negativeMarks"
    FROM "TestPattern"
    WHERE "id" = NEW."testPatternId"
    RETURNING "id" INTO backup_temp_test_id;

    -- Update BatchTest with the IDs of the created TempTest rows
    UPDATE "BatchTest"
    SET "tempTestId" = temp_test_id,
        "backupTempTestId" = backup_temp_test_id
    WHERE "id" = NEW."id";

    RETURN NEW;
END;
$$;

ALTER FUNCTION create_temp_test_entries() OWNER TO neetprep_rw;
