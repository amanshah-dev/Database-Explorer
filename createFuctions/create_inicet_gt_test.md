CREATE FUNCTION create_inicet_gt_test(
    testquestiondomain integer[] DEFAULT ARRAY[2123625, 2123626, 2123627, 2123628, 2123630, 2123629, 2123631, 2123641, 2123632, 2123642, 2123640, 2123639, 2123636, 2123637, 2123635, 2123634, 2123643, 2123633], 
    questions_remaining integer[] DEFAULT ARRAY[15, 10, 12, 20, 20, 15, 12, 5, 2, 2, 4, 5, 15, 20, 20, 15, 1, 7], 
    total_questions integer DEFAULT 200, 
    question_tag character varying DEFAULT 'INICET'::character varying
) RETURNS void
    LANGUAGE plpgsql
AS
$$
DECLARE
    new_test_id INT;
    question_record RECORD;
    test_position INT;
    test_name VARCHAR;
    total_questions_inserted INT := 0;
    question_count INT;
    AlreadyCreatedTest INT[];
    domain_mapping INT[];  -- Declare domain_mapping as variable
    inserted_question_ids INT[] := '{}';  -- Array to track already inserted questions
    sum_questions INT;
    new_question_id INT;  -- To store the ID of the newly created duplicate question
    positive_marks INT;
    negative_marks INT;
BEGIN
    -- Step 1: Assign the input testquestiondomain array to domain_mapping
    domain_mapping := testquestiondomain;

    -- Step 2: Calculate the sum of questions_remaining
    sum_questions := (SELECT sum(x) FROM unnest(questions_remaining) AS x);

    RAISE NOTICE 'Sum of questions_remaining: %', sum_questions;

    -- Step 3: Validate that total_questions equals the sum of questions_remaining
    IF total_questions != sum_questions THEN
        RAISE EXCEPTION 'Total number of questions (%s) does not match the sum of questions_remaining (%s). Aborting test creation.', total_questions, sum_questions;
    END IF;

    -- Step 4: Find existing tests for the passed tag (NEETPG, INICET, FMGE)
    EXECUTE format('SELECT ARRAY(SELECT "id" FROM "Test" WHERE "name" LIKE ''%s GT%%'')', question_tag)
    INTO AlreadyCreatedTest;

    RAISE NOTICE 'AlreadyCreatedTest test IDs: %', AlreadyCreatedTest;

    -- Step 5: Determine new test position based on the tag
    EXECUTE format('SELECT COUNT(*) + 1 FROM "Test" WHERE "name" LIKE ''%s GT%%''', question_tag)
    INTO test_position;

    test_name := question_tag || ' GT - ' || test_position;  -- Create test name based on the tag
    RAISE NOTICE 'Creating new test: %', test_name;

    -- Step 6: Determine the marking scheme based on the question tag
    IF question_tag = 'INICET' THEN
        positive_marks := 3;
        negative_marks := 1;
    ELSE
        positive_marks := 4;
        negative_marks := 1;
    END IF;

    -- Step 7: Insert into Test table
    INSERT INTO "Test"("name", "positiveMarks", "negativeMarks", "creatorId", "createdAt", "updatedAt", "numQuestions")
    VALUES (test_name, positive_marks, negative_marks, 1, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, total_questions)
    RETURNING "id" INTO new_test_id;

    RAISE NOTICE 'New test created with ID: %', new_test_id;

    -- Step 8: Insert questions domain-wise
    FOR i IN 1..array_length(domain_mapping, 1) LOOP
        question_count := 0;

        FOR question_record IN
            SELECT q."id", tq."testId"
            FROM "TestQuestion" tq
            JOIN "Test" t ON tq."testId" = t."id"
            JOIN "Question" q ON tq."questionId" = q."id"
            JOIN "QuestionTag" qt ON q."id" = qt."questionId"
            JOIN "Tag" tg ON qt."tagId" = tg."id"
            WHERE t."id" = domain_mapping[i]
            AND tg."tag" = question_tag
            AND q."id" NOT IN (
                SELECT tq2."questionId"
                FROM "TestQuestion" tq2
                JOIN "Test" t2 ON tq2."testId" = t2."id"
                WHERE t2."name" LIKE question_tag || ' GT%'
                AND t2."id" = ANY (AlreadyCreatedTest)
            )
            -- Exclude both the original question and its duplicates
            AND q."orignalQuestionId" IS NULL  -- Exclude questions that are already duplicates
            AND NOT EXISTS (
                SELECT 1
                FROM "Question" q2
                WHERE q2."orignalQuestionId" = q."id"  -- Exclude duplicates of the current question
            )
            LIMIT questions_remaining[i]
        LOOP
            -- Step 9: Insert duplicate question and mark as duplicate
            INSERT INTO "Question"("question", "options", "correctOptionIndex", "explanation", "createdAt", "updatedAt", "creatorId", "canvasQuestionId", "canvasQuizId", "deleted", "type", "paidAccess", "explanationMp4", "level", "jee", "sequenceId", "proofRead", "topicId", "subjectId", "ncert", "correctOptions", "sprinkler", "lockCount", "published", "orignalQuestionId")
            SELECT
                q."question",
                q."options",
                q."correctOptionIndex",
                q."explanation",
                CURRENT_TIMESTAMP,
                CURRENT_TIMESTAMP,
                q."creatorId",
                q."canvasQuestionId",
                q."canvasQuizId",
                q."deleted",
                q."type",
                q."paidAccess",
                q."explanationMp4",
                q."level",
                q."jee",
                q."sequenceId",
                q."proofRead",
                q."topicId",
                q."subjectId",
                q."ncert",
                q."correctOptions",
                q."sprinkler",
                q."lockCount",
                q."published",
                q."id"  -- This sets the original question as the duplicate's parent
            FROM "Question" q
            WHERE q."id" = question_record."id"
            RETURNING "id" INTO new_question_id;

            -- Step 10: Insert the new duplicate question into TestQuestion
            INSERT INTO "TestQuestion"("testId", "questionId", "createdAt", "updatedAt", "seqNum")
            VALUES (new_test_id, new_question_id, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, total_questions_inserted + 1);

            -- Track inserted question
            inserted_question_ids := array_append(inserted_question_ids, new_question_id);

            total_questions_inserted := total_questions_inserted + 1;
            question_count := question_count + 1;
        END LOOP;

        -- Log shortfall if any
        IF question_count < questions_remaining[i] THEN
            RAISE NOTICE 'Could not insert enough questions for domain ID: % (required: %, inserted: %)',
                domain_mapping[i], questions_remaining[i], question_count;
        ELSE
            RAISE NOTICE 'Inserted % questions for domain ID: %', question_count, domain_mapping[i];
        END IF;
    END LOOP;

    -- Step 11: If not enough total questions, rollback
    IF total_questions_inserted < total_questions THEN
        RAISE NOTICE '% with name "%", was not able to provide enough unique questions (needed %, found %). Test creation cancelled.',
            test_name, test_name, total_questions, total_questions_inserted;
        DELETE FROM "Test" WHERE "id" = new_test_id;
        RETURN;
    END IF;

    -- Step 12: Final update
    UPDATE "Test"
    SET "numQuestions" = total_questions_inserted
    WHERE "id" = new_test_id;

    RAISE NOTICE 'Test ID: % now has % questions', new_test_id, total_questions_inserted;
END;
$$;

ALTER FUNCTION create_inicet_gt_test(integer[], integer[], integer, varchar) OWNER TO learner;
