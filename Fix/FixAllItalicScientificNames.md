CREATE FUNCTION "FixAllItalicScientificNames"(test_id INTEGER) RETURNS VOID
    LANGUAGE plpgsql
AS
$$
BEGIN
    UPDATE "Question" 
    SET "question" = REPLACE("question", (' ' || "word"), (' <i>' || "word" || '</i>')), "updatedAt" = CURRENT_TIMESTAMP
    FROM "ItalicsWord"
    WHERE "question" ~ (' ' || "word") 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (test_id));

    UPDATE "Question" 
    SET "question" = REPLACE("question", ("word" || ' '), ('<i>' || "word" || '</i> ')), "updatedAt" = CURRENT_TIMESTAMP
    FROM "ItalicsWord"
    WHERE "question" ~ ("word" || ' ') 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (test_id));

    UPDATE "Question" 
    SET "question" = REPLACE("question", ('&nbsp;' || "word"), (' <i>' || "word" || '</i>')), "updatedAt" = CURRENT_TIMESTAMP
    FROM "ItalicsWord"
    WHERE "question" ~ ('&nbsp;' || "word") 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (test_id));

    UPDATE "Question" 
    SET "question" = REPLACE("question", ("word" || '&nbsp;'), ('<i>' || "word" || '</i> ')), "updatedAt" = CURRENT_TIMESTAMP
    FROM "ItalicsWord"
    WHERE "question" ~ ("word" || '&nbsp;') 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (test_id));

    UPDATE "Question" 
    SET "question" = REPLACE("question", ('<td>' || "word" || '</td>'), ('<td><i>' || "word" || '</i></td>')), "updatedAt" = CURRENT_TIMESTAMP
    FROM "ItalicsWord"
    WHERE "question" ~ ('<td>' || "word" || '</td>') 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (test_id));

    UPDATE "Question" 
    SET "question" = REPLACE("question", ("word" || ', '), ('<i>' || "word" || '</i>, ')), "updatedAt" = CURRENT_TIMESTAMP
    FROM "ItalicsWord"
    WHERE "question" ~ ("word" || ', ') 
    AND "id" IN (SELECT "questionId" FROM "TestQuestion" WHERE "testId" IN (test_id));
END;
$$;

ALTER FUNCTION "FixAllItalicScientificNames"(INTEGER) OWNER TO learner;
