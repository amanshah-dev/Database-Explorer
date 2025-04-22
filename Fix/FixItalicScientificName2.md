CREATE FUNCTION "FixItalicScientificName2"(name TEXT) RETURNS VOID
    LANGUAGE plpgsql
AS
$$
BEGIN
    UPDATE "Question" 
    SET "question" = REPLACE("question", (' ' || name), (' <i>' || name || '</i>')), 
        "updatedAt" = CURRENT_TIMESTAMP 
    WHERE "question" ~ (' ' || name) 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (1910529));

    UPDATE "Question" 
    SET "question" = REPLACE("question", (name || ' '), ('<i>' || name || '</i> ')), 
        "updatedAt" = CURRENT_TIMESTAMP 
    WHERE "question" ~ (name || ' ') 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (1910529));

    UPDATE "Question" 
    SET "question" = REPLACE("question", ('&nbsp;' || name), (' <i>' || name || '</i>')), 
        "updatedAt" = CURRENT_TIMESTAMP 
    WHERE "question" ~ ('&nbsp;' || name) 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (1910529));

    UPDATE "Question" 
    SET "question" = REPLACE("question", (name || '&nbsp;'), ('<i>' || name || '</i> ')), 
        "updatedAt" = CURRENT_TIMESTAMP 
    WHERE "question" ~ (name || '&nbsp;') 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (1910529));

    UPDATE "Question" 
    SET "question" = REPLACE("question", ('<td>' || name || '</td>'), ('<td><i>' || name || '</i></td>')), 
        "updatedAt" = CURRENT_TIMESTAMP 
    WHERE "question" ~ ('<td>' || name || '</td>') 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (1910529));
END
$$;

ALTER FUNCTION "FixItalicScientificName2"(TEXT) OWNER TO learner;
