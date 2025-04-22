-- Create function to check conditions based on ncertSentenceId for a trigger
create function check_conditions_based_on_ncertsentenceid() returns trigger
    language plpgsql
as
$$
BEGIN
  -- Check if "ncertSentenceId" is present
  IF NEW."ncertSentenceId" IS NOT NULL THEN
    -- Check if "userId" and "noteId" are present
    IF NEW."userId" IS NULL OR NEW."noteId" IS NULL THEN
      RAISE EXCEPTION 'userId and noteId must be present when ncertSentenceId is not null';
    END IF;

    -- Check if "details" contains "startDataUid" and "endDataUid"
    IF NOT (NEW."details"->>'startDataUid' IS NOT NULL AND NEW."details"->>'endDataUid' IS NOT NULL) THEN
      RAISE EXCEPTION 'details must contain startDataUid and endDataUid when ncertSentenceId is not null';
    END IF;
  END IF;

  RETURN NEW;
END;
$$;

-- Alter function owner
alter function check_conditions_based_on_ncertsentenceid() owner to neetprep_rw;
