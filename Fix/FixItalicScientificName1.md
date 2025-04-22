CREATE FUNCTION "FixItalicScientificName1"(name TEXT) RETURNS VOID
    LANGUAGE plpgsql
AS
$$
BEGIN
    UPDATE "Question" 
    SET "question" = REPLACE("question", ('<td>' || name || '</td>'), ('<td><i>' || name || '</i></td>')), 
        "updatedAt" = CURRENT_TIMESTAMP
    WHERE "question" ~ ('<td>' || name || '</td>');
END
$$;

ALTER FUNCTION "FixItalicScientificName1"(TEXT) OWNER TO learner;
