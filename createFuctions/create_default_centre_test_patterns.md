CREATE FUNCTION create_default_centre_test_patterns() RETURNS TRIGGER
    LANGUAGE plpgsql
AS
$$
BEGIN
    INSERT INTO "CentreTestPattern" ("centreId", "testPatternId")
    SELECT NEW."id", "TestPattern"."id" 
    FROM "TestPattern"
    WHERE "TestPattern"."id" IN (134, 136, 3, 4)
    ON CONFLICT ("centreId", "testPatternId") DO NOTHING;
    
    RETURN NEW;
END;
$$;

ALTER FUNCTION create_default_centre_test_patterns() OWNER TO neetprep_rw;
