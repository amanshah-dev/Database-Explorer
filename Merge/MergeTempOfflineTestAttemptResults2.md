CREATE FUNCTION "MergeTempOfflineTestAttemptResults"(testid INTEGER, usetestid INTEGER, offlinetestattemptid INTEGER) RETURNS VOID
LANGUAGE plpgsql
AS
$$
BEGIN
  EXECUTE "CalculateTempOfflineTestAttemptResults"(testId, offlineTestAttemptId);
END
$$;

ALTER FUNCTION "MergeTempOfflineTestAttemptResults"(INTEGER, INTEGER, INTEGER) OWNER TO neetprep_rw;
