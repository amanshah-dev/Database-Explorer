CREATE FUNCTION "FixItalicScientificName"(name TEXT) RETURNS VOID
    LANGUAGE plpgsql
AS
$$
BEGIN
    UPDATE "Question" 
    SET "question" = REPLACE("question", (' ' || name), (' <i>' || name || '</i>')), "updatedAt" = CURRENT_TIMESTAMP
    WHERE "question" ~ (' ' || name);

    UPDATE "Question" 
    SET "question" = REPLACE("question", (name || ' '), ('<i>' || name || '</i> ')), "updatedAt" = CURRENT_TIMESTAMP
    WHERE "question" ~ (name || ' ');

    UPDATE "Question" 
    SET "question" = REPLACE("question", (' ' || name), (' <i>' || name || '</i>')), "updatedAt" = CURRENT_TIMESTAMP
    WHERE "question" ~ (' ' || name);

    UPDATE "Question" 
    SET "question" = REPLACE("question", (name || ' '), ('<i>' || name || '</i> ')), "updatedAt" = CURRENT_TIMESTAMP
    WHERE "question" ~ (name || ' ');

    UPDATE "Question" 
    SET "question" = REPLACE("question", ('&nbsp;' || name), (' <i>' || name || '</i>')), "updatedAt" = CURRENT_TIMESTAMP
    WHERE "question" ~ ('&nbsp;' || name);

    UPDATE "Question" 
    SET "question" = REPLACE("question", (name || '&nbsp;'), ('<i>' || name || '</i> ')), "updatedAt" = CURRENT_TIMESTAMP
    WHERE "question" ~ (name || '&nbsp;');

    UPDATE "Question" 
    SET "question" = REPLACE("question", ('&nbsp;' || name), (' <i>' || name || '</i>')), "updatedAt" = CURRENT_TIMESTAMP
    WHERE "question" ~ ('&nbsp;' || name);

    UPDATE "Question" 
    SET "question" = REPLACE("question", (name || '&nbsp;'), ('<i>' || name || '</i> ')), "updatedAt" = CURRENT_TIMESTAMP
    WHERE "question" ~ (name || '&nbsp;');
END
$$;

ALTER FUNCTION "FixItalicScientificName"(TEXT) OWNER TO learner;
