CREATE FUNCTION "FixItalicScientificName3Last"(name TEXT) RETURNS VOID
    LANGUAGE plpgsql
AS
$$
BEGIN
    UPDATE "Question" 
    SET "question" = REPLACE("question", (name || ', '), ('<i>' || name || '</i>, ')), 
        "updatedAt" = CURRENT_TIMESTAMP 
    WHERE "question" ~ (name || ', ') 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (2039057));
END
$$;

ALTER FUNCTION "FixItalicScientificName3Last"(TEXT) OWNER TO learner;
