create function copy_and_delete_answers_by_section(_userid integer, _testid integer, _section text) returns void
    language plpgsql
as
$$
BEGIN
    WITH section_ranges AS (
        /* Extract sections and compute start and end numbers */
        SELECT 
            ordinality AS "sectionIndex",
            value->>0 AS "section",
            COALESCE((value->>1)::integer, 1) AS "startNum",
            COALESCE(
                (LEAD((value->>1)) OVER (ORDER BY ordinality))::integer, 
                "Test"."numQuestions" + 1
            ) - 1 AS "endNum"
        FROM "Test"
        LEFT JOIN LATERAL json_array_elements(COALESCE("Test"."sections", '[]'::json)) 
            WITH ORDINALITY 
            ON TRUE
        WHERE "Test"."id" = _testId
    ),
    ordered_questions AS (
        /* Assign a unique question number based on seqNum and questionId */
        SELECT
            "TestQuestion"."questionId",
            "Question"."correctOptionIndex",
            ROW_NUMBER() OVER (
                ORDER BY "TestQuestion"."seqNum" ASC, "TestQuestion"."questionId" ASC
            ) AS "questionNum"
        FROM "TestQuestion"
        JOIN "Question" 
            ON "Question"."id" = "TestQuestion"."questionId"
        WHERE "TestQuestion"."testId" = _testId
    ),
    question_sections AS (
        /* Assign each question to a section based on questionNum */
        SELECT 
            oq."questionId",
            oq."correctOptionIndex",
            sr."section"
        FROM ordered_questions oq
        JOIN section_ranges sr
            ON oq."questionNum" BETWEEN sr."startNum" AND sr."endNum"
        WHERE sr."section" = _section  -- Filter for the specific section
    ),
    -- Fetch the user's answers that belong to the specified section and store them in a CTE
    answers_for_user AS (
        SELECT 
            a."id",  -- Answer ID to be used for deletion
            a."questionId",
            a."userAnswer",
            a."testAttemptId",
            a."durationInSec",
            a."userId",
            a."createdAt",
            a."updatedAt"
        FROM "Answer" a
        JOIN question_sections qs
            ON a."questionId" = qs."questionId"
        WHERE a."userId" = _userId
    ),
    -- Insert the selected answers into the CopyAnswer table using a CTE
    insert_into_copyanswer AS (
        INSERT INTO "CopyAnswer" (
            "questionId", "userId", "testAttemptId", "durationInSec", "userAnswer", "createdAt", "updatedAt"
        )
        SELECT 
            "questionId",
            "userId",
            "testAttemptId",
            "durationInSec",
            "userAnswer",
            "createdAt",
            "updatedAt"
        FROM answers_for_user
        RETURNING 1  -- Return dummy value to satisfy the CTE
    )
    -- Now delete the answers from the Answer table
    DELETE FROM "Answer"
    WHERE "id" IN (
        SELECT "id" FROM answers_for_user
    );
END;
$$;

alter function copy_and_delete_answers_by_section(integer, integer, text) owner to neetprep_rw;
