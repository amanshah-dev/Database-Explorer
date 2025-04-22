CREATE FUNCTION get_chapter_section_test_ids(chap_id INTEGER, sect section_type, test_pattern_id INTEGER) 
    RETURNS INTEGER[] 
    LANGUAGE plpgsql
AS
$$
DECLARE
    test_ids INTEGER[];
BEGIN
    -- Fetching test IDs ordered by seqId from QuesBankTest table,
    -- assuming it connects test IDs with chapters and sections.
    SELECT array_agg("testId" ORDER BY "seqId")
    INTO test_ids
    FROM "QuesBankTest"
    WHERE "chapterId" = chap_id 
      AND "section" = sect 
      AND "testPatternId" = test_pattern_id;

    RETURN test_ids;
END;
$$;

ALTER FUNCTION get_chapter_section_test_ids(integer, section_type, integer) OWNER TO neetprep_rw;
