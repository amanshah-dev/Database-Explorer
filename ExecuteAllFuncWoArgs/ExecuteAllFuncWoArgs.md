CREATE FUNCTION "ExecuteAllFuncWoArgs"() RETURNS VOID
    LANGUAGE plpgsql
AS
$$
BEGIN
  PERFORM "addComplimentaryJumboForCTS"();
END
$$;

ALTER FUNCTION "ExecuteAllFuncWoArgs"() OWNER TO neetprep_rw;
