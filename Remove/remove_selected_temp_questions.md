create function remove_selected_temp_questions(temptestid integer)
    returns TABLE("questionId" integer)
    language plpgsql
as
$$
BEGIN
    RETURN QUERY
    SELECT "TempTestQuestion"."questionId"
    FROM "TempTestQuestion"
    WHERE "tempTestId" = tempTestId
    UNION
    SELECT "TestQuestion"."questionId"
    FROM "BatchTest" bt1
    JOIN "Batch"
        ON bt1."tempTestId" = tempTestId
        AND bt1."batchId" = "Batch"."id"
    JOIN "BatchTest" bt2
        ON bt1."batchId" = bt2."batchId"
    JOIN "TestQuestion"
        ON bt2."testId" = "TestQuestion"."testId";
END;
$$;

alter function remove_selected_temp_questions(integer) owner to neetprep_rw;
