create function remove_user_attempted_questions(p_userid integer)
    returns TABLE("questionId" integer)
    language plpgsql
as
$$
BEGIN
    RETURN QUERY
    SELECT DISTINCT "Answer"."questionId"
    FROM "Answer"
    WHERE "userId" = p_userid
    AND "createdAt" >= NOW() - INTERVAL '1 year';
END;
$$;

alter function remove_user_attempted_questions(integer) owner to neetprep_rw;
