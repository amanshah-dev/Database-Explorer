create function test_func(p_id integer DEFAULT NULL::integer, p_text text DEFAULT NULL::text) returns void
    language plpgsql
as
$$
BEGIN
    RAISE NOTICE 'Called test_func(p_id=%, p_text=%)', p_id, p_text;
END;
$$;

alter function test_func(integer, text) owner to learner;
