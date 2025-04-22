CREATE FUNCTION "MergeTests"(intotestid INTEGER, fromtestids INTEGER[]) RETURNS VOID
LANGUAGE plpgsql
AS
$$
DECLARE
  testId INTEGER;
BEGIN
  FOREACH testId IN ARRAY fromTestIds
  LOOP
    PERFORM "MergeTests"(intoTestId, testId);
  END LOOP;
END
$$;

ALTER FUNCTION "MergeTests"(INTEGER, INTEGER[]) OWNER TO learner;
